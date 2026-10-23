# Gate contract

## One question, two layers

The gate is one question a runner asks before it starts anything: may I? It answers from two layers that never mix. The
**breakers** — the runner's own brake, the hub's brake, and the per-chunk spend cap — keep the code they have today:
`RunnerBrakes` (`runner/throttle/pause.py:157`) read by `Spawner.brakes` (`runner/lifecycle/spawn.py:214`), the
ceiling's `SpendCeiling` step first in the tick (`runner/loop/steps.py:134`, `runner/loop/tick.py:89`), and the chunk
cap checked between attempts (`runner/lifecycle/drain.py:201`). Each still trips and stays tripped until a person clears
it. The **throttles** are new: the subscription floor and the four levers, each a pure answer over readings the runner
already holds, none of them writing anything when it says no.

Breakers are asked first, and a tripped one ends the question: a braked runner is braked, whatever its throttles would
say, and nothing the hub shows of it calls it declining ([surfaces.md](./surfaces.md)). The throttles govern only a
runner no breaker holds.

## Where it lives

The gate is the `throttle` package's, beside the brakes it ranks under (`bzh:domain-package-layers`):

- `runner/throttle/settings.py` — `ThrottleSettings`, the configured levers ([configuration.md](./configuration.md)).
- `runner/throttle/levers.py` — one pure function per lever, each taking its settings, its readings, and `now`, and
  returning a `LeverAnswer`: the lever's name, `admits`, the reading it judged in operator words, and, when it says no,
  `until`, the soonest instant at which the same readings would say yes, when one can be known
  ([levers.md](./levers.md)).
- `runner/throttle/floor.py` — the subscription floor's answer, a `FloorAnswer` over the subscription's exhaustion
  ([exhaustion.md](./exhaustion.md)).
- `runner/throttle/gate.py` — `GateVerdict`, the floor's answer and every configured lever's together, deciding by value
  (`bzh:domain-orchestration-split`), and `ThrottleGate`, the step module that gathers the readings and asks for the
  verdict, behind its own context Protocol, `GateContext`, with its `_conforms_to_gate` sentinel in
  `runner/loop/context.py` (`bzh:narrow-seams`).

`GateVerdict` answers three questions, one per kind of start:

| Property              | Says yes when                                | Asked by                                        |
| --------------------- | -------------------------------------------- | ----------------------------------------------- |
| `admits_claim`        | the floor and every configured lever say yes | FILL, before it peeks                           |
| `admits_step`         | the floor and every configured lever say yes | every mint of a fresh lease for a node-step     |
| `admits_continuation` | the floor says yes                           | every start that continues a step already begun |

A claim and a step ask the same thing; they are named apart because they are asked from different places, and a later
lever may come to tell them apart. A continuation answers to the floor alone: a step the gate admitted is allowed to
finish — its answered ask woken, its overload backoff resumed, its verdict judged — because the line the plan draws is
before each step, not inside one. The floor is the exception because a start against an exhausted subscription meets the
wall the moment it runs, whatever kind of start it is.

`GateVerdict.declining` is every answer saying no, the floor's first and then the levers' in a fixed order — pace,
utilization, budget, schedule — so the runner can say every reason it is waiting.

## What it reads

`ThrottleGate` reads, each time it is asked, only what the configured levers need:

- the runner's `ThrottleSettings` and the slug and sample interval of the one subscription it names, if any;
- the newest sample of that slug (`IReadExternalUsageRepository.latest_external_usage_windows`,
  `runner/usage/repository.py:369`), one indexed read, when the floor or a reading lever needs it;
- the newest usage-limit mark and the newest hand clear of that subscription, or the runner's own when it names none,
  two indexed reads ([exhaustion.md](./exhaustion.md) §The mark, §Clearing by hand);
- the runner's spend over the budget's window (`IReadUsageRepository.usage_since`, `runner/usage/repository.py:298`),
  one aggregate read, only when a budget is configured;
- the clock.

Nothing is cached across asks. Each ask is a few indexed reads (`bzh:read-cost`), and the answer must see a usage limit
marked a moment earlier on the same host by a neighbour spending the same plan — which a verdict held from the start of
the tick would not.

## Every start asks it

**Claims.** `Fill.run` (`runner/loop/steps.py:470`–`481`) today returns early on `brakes.blocks_claims`. Its early
return becomes one call into the gate's step module, `ThrottleGate.claims(brakes)`, which returns the brakes' refusal,
the gate's declining verdict, or admission, and FILL branches on that one result (`bzh:narrow-seams`). A declining
runner peeks, acquires, and claims nothing, and logs once per tick that it is declining and on which levers, as the
braked one logs today (`:474`). `InterruptedClaims.reconcile` still runs ahead of the check (`:472`). The `braked` it is
told decides one thing only, whether a binding whose claim never landed claims its still-`ready` chunk again
(`interrupted_claim_move`, `runner/lifecycle/model.py:333`–`334`), and that is a claim like any other, so it is told the
runner is held back while a brake blocks claims or the gate declines one. Its other arms are unchanged; the adopt arm's
start asks the gate as every start does, below.

**Starts.** Every process start already funnels through `Spawner.suppressed` (`runner/lifecycle/spawn.py:219`), and
`tests/test_spawn_suppressed_registry.py` holds every start site to calling it. `suppressed` gains a required keyword,
`start`, a `StartKind` of `STEP`, `CONTINUATION`, or `ESCALATION`, and asks the gate after the brakes and the chunk's
own pause have said nothing:

- `STEP` — `Spawner.spawn` (`:267`), the funnel every fresh lease passes through: a claim's first entry, a next node
  entered from an apply response or from FILL's adopt arm, a retry's requeue (`runner/lifecycle/attempt.py:206`), and a
  restart's re-entry. Asks `admits_step`.
- `CONTINUATION` — the restart-resume, the answer wake, the unpause wake, the overload backoff's resume, and the judge's
  launch and relaunch (`runner/lifecycle/dormant.py`, `runner/lifecycle/judgement/judgement.py`). Asks
  `admits_continuation`.
- `ESCALATION` — the two zero-budget mints that exist only to escalate a chunk (`spawn.py:449`, `:479`). They spend
  nothing, so they ask the breakers alone, as they do today.

The registry test gains the rule that every gate call names its `start`, so a new start site cannot slip in under a
default. A sibling, `Spawner.suppression`, returns the cause — braked, chunk paused, or declined with its verdict — for
the two callers that act on it ([release.md](./release.md)); `suppressed` stays the boolean every other site reads.

## The line holds mid-graph

A declined start writes nothing, which is what already makes a suppressed start safe: `Spawner.spawn` is always its
caller's last statement (`runner/lifecycle/spawn.py:264`–`266`), and every path that reaches it recovers on a later
tick. A chunk whose next node the gate declines therefore parks between steps without a park of its own: its route,
binding, environments, and session stay as they were, no lease is minted, so no attempt is counted against its retries
(`runner/lifecycle/attempt.py:111`), and each tick re-enters its node through the path that owns that entry today —
`HeldChunk.drive`'s `SPAWN_ADVANCED` (`runner/lifecycle/held_chunk.py:110`) or FILL's adopt
(`runner/lifecycle/claim.py:404`), as `owns_node_entry` (`runner/lifecycle/model.py:265`) decides. Each re-entry asks
the gate again, and the first that is admitted spawns the node into its warm environment under its declared session, as
a chunk held over a per-chunk pause is entered today (`adopt_enters_node`, `runner/lifecycle/model.py:348`).

A lease that meets an outright usage limit mid-step parks in place as it does now ([exhaustion.md](./exhaustion.md) §A
lease limited mid-step); what changes is what wakes it, and, on a runner set to release such a lease, what lets it go
([release.md](./release.md) §Releasing a usage-limited lease).

## What it leaves

Declining leaves no fact, no row, and no outbound report. What the gate is saying now is derived on every read from the
same readings, never stored: the runner's status and its registration carry the verdict as of the moment they are read
or sent ([surfaces.md](./surfaces.md)), and the runner log carries one line per tick while it declines.
