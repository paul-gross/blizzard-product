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
- **An exhausted window** — a window a sample showed at 100 percent, read as `exhausted_pending` does while its reset is
  still ahead (`ExternalSubscriptionUsageWindow.exhausted_pending`, `runner/subscriptions/subscription_sampler.py:60`),
  and held past that reset until a sample shows it turned over (§What opens a hold). This is what holds a subscription
  another host's runners ran dry, or one whose harness ran it dry without reporting it.

Each hold has an **expected reset**. A mark's is the reset the harness named (`UsageLimit.resets_at`,
`runner/harness/usage.py:65`). A mark naming none takes the soonest pending reset of an exhausted window in the first
sample taken at or after it — the reading `soonest_exhausted_reset` already makes (`runner/usage/repository.py:188`),
read for the one slug only — and, until such a sample exists or when it shows no exhausted window, thirty minutes after
the mark, after which the next start is the probe: a limit met again marks again. An exhausted window's expected reset
is its own `resets_at`.

## What opens a hold

A hold opens on evidence that its window has **turned over**, never on a reading that is merely under the line. Within
one period a provider's counter only rises, and it can lag the harness that has just been refused: a limit met at what
the provider still reports as 99 percent is met all the same, and a sample at 99 percent minutes later — or hours later,
in the same period — is the lag, not an opening. Clearing on it would start a runner straight back into the wall on
every sample. A window that has turned over reports a period later than the one the hold was read in — its `resets_at`
is later — and stands under 100 percent in it.

Each hold has a **held window**. An exhausted window's is that window, in the period it was read at 100 percent. A
mark's is the window whose `resets_at`, in the newest sample at or before `marked_at`, is the reset the harness named,
to the minute; a mark naming no reset, or a reset no window carries, has no held window, and holds every window the
sample reports.

- **Before the expected reset**, a sample opens a hold only by showing its held window turned over. An owner who buys or
  redeems a reset early starts a new period, and the floor follows within one sample interval rather than holding the
  fleet until the date the harness named; the subscription goes on being sampled on its cadence while it is held, which
  is what notices an early reset at all. A mark with no held window is never opened early by a sample: what opens it
  early is an operator's hand (§Clearing by hand).
- **After the expected reset**, a sample taken since the reset opens the hold when every window it reports stands under
  100 percent in a period still running when it was sampled. A window still reporting a period whose reset has passed is
  a provider that has not rolled its counter yet, and is no evidence either way: the hold waits for the next sample. A
  window reporting its new period at 100 percent is a subscription exhausted again, and stands as an exhausted window
  under its own reset.

The expected reset is the hold's best guess at when it lifts — the figure the surfaces show, and the moment the resample
below fires — and decides alone only where no sample can confirm: a subscription whose provider binds no sampler, a
runner naming none, and a mark naming no reset before any sample exists. While a hold stands past its expected reset the
floor reads `reset passed, awaiting a sample showing it open` ([surfaces.md](./surfaces.md)).

Evidence can also fail to come. A sampler whose credentials have lapsed, or a provider whose usage endpoint is down,
records misses and never a sample, and a provider slow to roll its counter reports a passed period sample after sample;
a hold that waited on either would stand as long as the outage did, with every runner of the plan behind it. So a hold
whose expected reset passed more than an hour ago, with no sample since that reset that is evidence either way, clears
on the clock: the **grace**, one hour, fixed (`FLOOR_UNCONFIRMED_GRACE`, `runner/throttle/floor.py`). The next start is
the probe, as it is for a mark naming no reset: a limit met again marks again, under whatever reset the harness names
that time. The grace answers a sampler that cannot see, never one that sees the window shut: a window reporting its new
period at 100 percent is a fresh hold, and the grace does not reach it. While the grace runs the floor reads
`reset passed, no sample since — opens at 15:00Z`.

The derivation is a pure function in `runner/throttle/floor.py` over the newest mark, the newest sample, the newest hand
clear (§Clearing by hand), and `now`, returning a `FloorAnswer`: whether it admits, the slug, the hold that stands — a
mark naming its harness, or a window naming its label — its expected reset, and, when the floor stands open only because
of a clear, that clear. Nothing about exhaustion is stored as a flag (`bzh:facts-not-status`): the answer is derived
from the three newest rows on every ask. The newest mark supersedes every older one, since a newer report from a harness
is a fresher reading of the same subscription than an older one, so the read is newest-per-key
(`bzh:newest-per-key-read`).

## The mark

`subscription_usage_limits` is a new host-scoped table of the host store, declared beside `external_usage_samples`
(`runner/store/schema.py:799`) with `info={"scope": "host", "transfer": HOST_TRANSFER_APPEND}`, so ingest merges it by
slug as it merges samples. A row holds the slug, `marked_at`, the harness-named `resets_at` or none, the held window's
label and the `resets_at` of its period as the newest sample showed them when the mark was written, or none (§What opens
a hold), the harness id, the harness's own `detail`, and the invocation it was classified from — lease id, epoch,
generation, and whether a worker or a judge met it — unique on that invocation, so a replayed mark writes nothing. It is
reached only through a new `HostStores` member, `usage_limits`, in `runner/store/internal/host_usage_limit_store.py`
(`bzh:host-stores-confined`), whose read is the newest mark of a slug, served by an index on `(slug, marked_at)`. The
retention step prunes it with the samples (`runner/loop/steps.py:634`): a mark older than the longest window any
provider reports — eight days is ample — can no longer hold anything.

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

A usage limit asks for a fresh sample rather than waiting out the cadence, so the board's usage rows catch up with what
the harness has just found — a sample that cannot open the mark it follows unless its held window has turned over — and
a hold whose reset has passed is confirmed open without waiting for the next scheduled sample. The host's sampling pass
decides both, reading the newest mark beside the newest attempt it already reads (`ExternalUsageSamplePass._sample_one`,
`runner/usage/external_usage_sample.py:85`): a slug is due on its cadence as today (`sample_due`,
`runner/subscriptions/subscription_sampler.py:118`), **or** when its newest mark is newer than its newest attempt,
**or** when its standing hold's expected reset has passed with no attempt since. Each early sample fires once, since the
attempt it records is then the newest, and a sample that misses is an attempt all the same, so a provider that cannot be
reached falls back to the cadence rather than being asked every host tick. The pass runs on the host's own thread, as
now (`bzh:lane-contract`), so a resample never holds a runner's tick.

A subscription whose provider binds no sampler is never resampled and has no samples to confirm with: its marks clear at
their expected reset alone, as a runner naming no subscription's do.

## Clearing by hand

An operator can know better than every reading the floor has: a reset redeemed that the provider's counter has not
caught up with, a sampler reading the wrong account, a limit a harness misreported. `blizzard runner clear-floor` lifts
the holds standing, beside `pause` and `start` (`runner/cli/control.py:280`, `:309`) and taking `--by` as they do. It
names exactly one target:

- `--subscription <slug>` clears a subscription's floor for every runner of the host spending it, through
  `POST /api/host/subscriptions/{slug}/floor/clear` on the host's API, beside the subscriptions read the status verb
  already makes (`runner/cli/control.py:141`).
- `--runner <id>` clears the floor of a runner naming no subscription, through `POST /api/runner/floor/clear` on that
  runner's own API. Naming a runner that spends a subscription is refused with the slug to name instead, so nobody lifts
  a whole plan believing they lifted one machine.

Both routes reach `openapi/runner.openapi.json` when it is regenerated, and neither reaches the hub. The runner panel's
floor row offers the same clear ([surfaces.md](./surfaces.md) §The runner's panel).

A clear is a row: `subscription_floor_clears`, host-scoped beside `subscription_usage_limits` with the same transfer,
holding the slug, `cleared_at`, and `by`; and `runner_floor_clears`, its runner-scoped twin without the slug. Each is
reached through the same store member as its marks, read newest-per-key, and pruned with them by the retention step.
Neither is reported to the hub: the hub learns only that the runner stopped declining, on its next registration
([surfaces.md](./surfaces.md)).

A clear sets aside the holds standing when it was written, and nothing after them: every mark at or before `cleared_at`,
and every window the newest sample at or before `cleared_at` showed exhausted, for as long as later samples show that
same window at that same `resets_at`. A provider's counter still reading 100 percent for the window the operator cleared
does not undo the clear. New evidence holds the plan again: a mark after the clear — the harness meeting the limit, as
the next start will if the operator was wrong — or a window exhausted under a later reset, or one that was not exhausted
when the clear was made. A clear on an open floor writes its row and changes nothing.

Nothing a clear does touches a brake: the breakers keep their own clear, `blizzard runner start`, and a clear lifts a
throttle no one ever had to lift. `docs/deployment/control-verbs.md` names `clear-floor` beside `pause` and `start`.

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

That is the lease's course under `[runner.throttle] on_usage_limit = "hold"`, the default. Under `"release"`, each tick
the floor still holds it, `on_unpause` releases the chunk to the queue instead of waiting, at the price of the half-done
step; a refusal leaves it holding as above ([release.md](./release.md) §Releasing a usage-limited lease).

## The rule

`blizzard-context`'s `domain/execution/throttling.md` states the floor as the rule that replaces
`bzh:subscription-brake-reach`:

> **`bzh:subscription-floor-reach`.** A subscription's exhaustion holds back every runner of the host spending it and no
> other, and lifts for all of them together when it clears. No runner's brake is engaged by it, and nothing a person
> does is needed to lift it, though a person may.
>
> **Why.** The limit belongs to the plan and carries its own expiry: every runner spending it would meet it on its next
> start, a runner paid from another plan has no reason to stop, and a fact that expires should not wait for someone to
> clear it.
>
> **Detect.** A limit met by one runner while another of its host spending the same subscription keeps starting work; a
> runner on another subscription, or none, held back by a neighbour's limit; a runner's brake engaged by a usage limit;
> a held-back runner still waiting after a sample shows its held window turned over and open, whether or not the
> expected reset has come; a runner admitted on a sample under 100 percent in the same period the limit was met in; one
> still waiting an hour past its expected reset with no sample since; one held by a hold an operator cleared, with no
> mark and no newly exhausted window after the clear.

`domain/execution/pause.md` keeps the runner's own brake as the operator's and the ceiling's alone, and `hosts.md`
§Subscriptions points its last sentence at the new rule.

In `docs/deployment/spend.md`, §Usage-limit pause becomes §Usage limits: the mark, the floor every spender waits behind,
the expected reset, the clear on a fresh sample, the hour's grace when no sample comes, the clear by hand, and the
parked lease that resumes on its own — with its two sentences saying only an operator clears a usage-limit brake
removed. §External subscription usage no longer calls the sampled utilization advisory only: the floor and the reading
levers judge by it, while the hub still decides nothing by it.
