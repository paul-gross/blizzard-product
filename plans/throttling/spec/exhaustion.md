# Exhaustion contract

The subscription floor is the one throttle no operator configures: every runner spending a subscription declines new
work while that subscription is exhausted. Exhaustion is a reading of the subscription, held on the host beside its
samples and keyed by its slug as they are, so every runner of the host spending the plan reads the same answer and none
of them is braked. One account spent from two hosts is two subscriptions here, each with its own samples and its own
marks, and a limit met on one holds back nothing on the other.

## What holds a subscription

A subscription is exhausted while either of two holds stands:

- **A usage-limit mark** — the harness's own report, written the moment a limited exit is classified (§The mark). It is
  the freshest reading there is, so it outranks the newest sample: a sample taken minutes before at 94 percent does not
  open a subscription a harness has since found closed.
- **An exhausted window** — a window of the newest sample at 100 percent with its reset still ahead
  (`ExternalSubscriptionUsageWindow.exhausted_pending`, `runner/subscriptions/subscription_sampler.py:60`). This is what
  holds a subscription another host's runners ran dry, or one whose harness ran it dry without reporting it.

Each hold has an **expected reset**. A mark's is the reset the harness named (`UsageLimit.resets_at`,
`runner/harness/usage.py:65`). A mark naming none takes the soonest pending reset of an exhausted window in the first
sample taken at or after it — the reading `soonest_exhausted_reset` already makes (`runner/usage/repository.py:188`),
read for the one slug only — and, until such a sample exists or when it shows no exhausted window, thirty minutes after
the mark, after which the next start is the probe: a limit met again marks again. An exhausted window's expected reset
is its own `resets_at`.

A hold clears once its expected reset has passed **and** a sample taken at or after that reset shows every window under
100 percent. The two conditions are the plan's: the reset having passed is not enough on its own, because a provider's
counter can lag its own reset, and a fresh sample alone is not enough, because a sample's lag behind the harness is
exactly what the mark exists to outrank. Until the sample arrives the floor reads
`reset passed, awaiting a fresh sample` ([surfaces.md](./surfaces.md)).

The derivation is a pure function in `runner/throttle/floor.py` over the newest mark, the newest sample, and `now`,
returning a `FloorAnswer`: whether it admits, the slug, the hold that stands — a mark naming its harness, or a window
naming its label — and its expected reset. Nothing about exhaustion is stored as a flag (`bzh:facts-not-status`): the
answer is derived from the two newest rows on every ask. The newest mark supersedes every older one, since a newer
report from a harness is a fresher reading of the same subscription than an older one, so the read is newest-per-key
(`bzh:newest-per-key-read`).

## The mark

`subscription_usage_limits` is a new host-scoped table of the host store, declared beside `external_usage_samples`
(`runner/store/schema.py:799`) with `info={"scope": "host", "transfer": HOST_TRANSFER_APPEND}`, so ingest merges it by
slug as it merges samples. A row holds the slug, `marked_at`, the harness-named `resets_at` or none, the harness id, the
harness's own `detail`, and the invocation it was classified from — lease id, epoch, generation, and whether a worker or
a judge met it — unique on that invocation, so a replayed mark writes nothing. It is reached only through a new
`HostStores` member, `usage_limits`, in `runner/store/internal/host_usage_limit_store.py` (`bzh:host-stores-confined`),
whose read is the newest mark of a slug, served by an index on `(slug, marked_at)`. The retention step prunes it with
the samples (`runner/loop/steps.py:634`): a mark older than the longest window any provider reports — eight days is
ample — can no longer hold anything.

A runner that names no subscription has no plan to mark, yet its harness's limit is just as real. Its marks go to
`runner_usage_limits`, a runner-scoped table of the same columns without the slug (`info={"scope": "runner"}`, through a
bound runner adapter, `bzh:runner-scoped-store`), and hold that runner alone. With no samples to read, its holds clear
at their expected reset.

## Marking a limit

Detection does not change: the adapter's `classify_usage_limit` (`runner/harness/adapter.py:368`) still translates
Claude Code's transcript `rate_limit` records (`runner/harness/claude_code/adapter.py:643`–`723`) and OpenCode's `429`
`error` events (`runner/harness/opencode/adapter.py:994`–`1021`) into a `UsageLimit`, after a worker generation exits
(`runner/loop/steps.py:584`) or a judge elicitation does (`runner/lifecycle/judgement/judgement.py:274`). What the loop
does with it changes. `_engage` in `runner/lifecycle/usage_limit.py` (`:137`) becomes `_mark`: it writes one mark — to
the runner's subscription, or to the runner itself when it names none — and the lease parks after it exactly as today
(`engage_and_park_worker`, `:87`; `engage_and_park_judge`, `:119`). No brake is engaged: a usage limit is a throttle and
never trips a breaker.

The mark is one row in one transaction, so the brake fan-out it replaces retires with everything only it used:
`UsageLimitBrakes` and `SpendingRunner` (`runner/throttle/pause.py:410`, `:418`), `usage_limit_reason` (`:199`),
`_fallback_reset` (`runner/lifecycle/usage_limit.py:153`), `UsageLimitContext.slug_spenders` and
`LoopContext.slug_spenders` (`runner/loop/context.py:200`), and `HostGraph.spenders_of` (`runner/composition.py:216`).
The runner's own brake keeps its two remaining engagers, the operator's pause and the ceiling's self-engagement
(`LocalBrake.engage`, `runner/throttle/pause.py:138`), and its locked transaction (`locked_brakes`, `:311`) is left with
the operator's pause as its only caller, over the one runner.

Across a crash the mark stands where the brake stood. The crash points become `usagelimit.worker-after-mark.before-park`
and `usagelimit.judge-after-mark.before-park` (`usage_limit.py:33`, `:39`), with the same meaning: a `kill -9` between
them leaves the subscription marked and the lease unparked, and the next pass classifies the exit again, replays the
mark as a no-op under its invocation key, and parks. The sweep's CI subset (`tests/crash/test_kill9_sweep.py:245`) names
the renamed worker point, and `architecture/crash-correctness/runner.md` §The usage-limit brake fan-out is rewritten as
the usage-limit mark: a **no-window** write followed by the park the existing points cover.

## The resample

A usage limit asks for a fresh sample rather than waiting out the cadence, so a gate does not go on trusting a sample
the harness has just contradicted, and a hold whose reset has passed is confirmed open without waiting for the next
scheduled sample. The host's sampling pass decides both, reading the newest mark beside the newest attempt it already
reads (`ExternalUsageSamplePass._sample_one`, `runner/usage/external_usage_sample.py:85`): a slug is due on its cadence
as today (`sample_due`, `runner/subscriptions/subscription_sampler.py:118`), **or** when its newest mark is newer than
its newest attempt, **or** when its standing hold's expected reset has passed with no attempt since. Each early sample
fires once, since the attempt it records is then the newest, and a sample that misses is an attempt all the same, so a
provider that cannot be reached falls back to the cadence rather than being asked every host tick. The pass runs on the
host's own thread, as now (`bzh:lane-contract`), so a resample never holds a runner's tick.

A subscription whose provider binds no sampler is never resampled and has no samples to confirm with: its marks clear at
their expected reset alone, as a runner naming no subscription's do.

## A lease limited mid-step

The lease whose worker or judge met the limit parks in place, its claim, route, epoch, and session intact, through
`Attempt.park_usage_limited` (`runner/lifecycle/attempt.py:439`) — the same `pause_parks` row an operator's per-chunk
pause leaves (`runner/store/schema.py:331`), now carrying the cause `usage-limit` so the two can be told apart
([release.md](./release.md) §Releasing a usage-limited lease). What wakes it changes. `DormantSession.on_unpause`
(`runner/lifecycle/dormant.py:231`) already waits while `Spawner.suppressed` says no (`:275`); that call names
`CONTINUATION` ([gate.md](./gate.md) §Every start asks it), so the park waits while the floor holds and wakes the
session — or relaunches the judge for a judge-side park (`_resume_judge_usage_limit_park`, `:455`) — on the first tick
the floor opens, with nobody at the keyboard. A chunk an operator also paused still waits for its own resume, as
`unpause_move` decides today. `park_usage_limited`'s docstring, which says the lease resumes once the brake lifts
(`attempt.py:445`–`446`), says it resumes once its floor opens.

That is the lease's course under `[runner.throttle] usage_limit_gate_type = "hold"`, the default. Under `"release"`,
each tick the floor still holds it, `on_unpause` releases the chunk to the queue instead of waiting, at the price of the
half-done step; a refusal leaves it holding as above ([release.md](./release.md) §Releasing a usage-limited lease).

## The rule

`blizzard-context`'s `domain/execution/throttling.md` states the floor as the rule that replaces
`bzh:subscription-brake-reach`:

> **`bzh:subscription-floor-reach`.** A subscription's exhaustion holds back every runner of the host spending it and no
> other, and lifts for all of them together when it clears. No runner's brake is engaged by it, and nothing a person
> does is needed to lift it.
>
> **Why.** The limit belongs to the plan and carries its own expiry: every runner spending it would meet it on its next
> start, a runner paid from another plan has no reason to stop, and a fact that expires should not wait for someone to
> clear it.
>
> **Detect.** A limit met by one runner while another of its host spending the same subscription keeps starting work; a
> runner on another subscription, or none, held back by a neighbour's limit; a runner's brake engaged by a usage limit;
> a held-back runner still waiting after the reset has passed and a fresh sample shows the window open.

`domain/execution/pause.md` keeps the runner's own brake as the operator's and the ceiling's alone, and `hosts.md`
§Subscriptions points its last sentence at the new rule.

In `docs/deployment/spend.md`, §Usage-limit pause becomes §Usage limits: the mark, the floor every spender waits behind,
the expected reset, the clear on a fresh sample, and the parked lease that resumes on its own — with its two sentences
saying only an operator clears a usage-limit brake removed. §External subscription usage no longer calls the sampled
utilization advisory only: the floor and the reading levers judge by it, while the hub still decides nothing by it.
