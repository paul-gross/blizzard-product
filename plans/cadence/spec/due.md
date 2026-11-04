# When an errand is due

Every rule here is a pure function in `hub/domain/cadence/`, of the cadence, the errand, the errand's last run, and one
instant `now`, read from the injected clock once per pass (`bzh:injected-clock`). Nothing is stored about when an errand
is next due: the answer is worked out each time it is asked, by the minting pass and by every read that shows it.

## The errands

A cadence's errands are worked out from its chores each time it is read, never stored. A chore's scopes resolve to its
errands as follows:

- **`all`** — every scope the routine sweeps now: `IReadRoutineScopeRepository.list_scopes` for the routine, less the
  project's `retired_slugs`. That is exactly `sweep_coverage`'s live set (`hub/domain/garden/runs/sweeps.py:123`), so a
  scope linked on Tuesday is an errand on the next look, and one unlinked or retired simply leaves the round.
- **A list** — one errand per listed slug, each standing or stopping on its own ([surfaces.md](./surfaces.md) §Needs
  attention).
- **Absent** — one errand over the routine's `default_scope_slug` (`hub/domain/garden/routines.py:268`).

An errand is identified by its chore's id and its scope's id. A scope's slug is unique in its project, so the two never
disagree.

## The last run

An errand's runs are the runs whose mark names its chore and scope ([mint.md](./mint.md) §The errand's mark), and its
last run is the newest of them by minted instant, whatever became of it. A run started by hand, or a run of the same
routine minted by another chore, carries no mark of this errand and is none of its runs. The read is one newest-per-key
query over `work_item_runs` for every errand of a tenant's cadences (`bzh:newest-per-key-read`), with the chunks' status
facts read in bulk beside it (`IReadChunkFactsRepository.status_facts_for`), never a read per errand.

A run whose chunk no longer exists is not a run. Deleting a minted chunk before any runner claims it, which
`CHUNK_VERB_LEGALITY` allows (`hub/domain/chunk/model.py:1032`), deletes its work item with it, and the errand falls
back on whatever run is then newest, or on the never-run rule below.

A last run is **unfinished** while its chunk's status is outside `TERMINAL_STATUSES` (`foundation/chunk_status.py:21`).
Blizzard has no failed status and no cancelled one, so the plan's three outcomes read as the chunk's own statuses:

- **Finished** is `done`.
- **Cancelled** is `stopped`. It is terminal, the spacing counts from it, and it is never retried.
- **Failed** is a run escalated to `needs_human`. It is not terminal, so the errand waits on it as it waits on any live
  run, and a failure is never quietly retried. Restarting it from there keeps the same chunk, and so the same mark and
  the same errand. Stopping it settles it as cancelled.

## One live run

`bzh:errand-one-live-run`: an errand whose last run is unfinished is never due. This is the first test, before any
moment is read, and the minting write re-establishes it under a lock ([mint.md](./mint.md) §The locked write). A fleet
behind on its queue therefore holds one run per errand, and an errand whose run waits on a person waits with it.

## The three anchors

A spacing counts from one moment of the last run, chosen by the cadence's `anchor`:

- **`scheduled`** — `Chunk.minted_at`.
- **`started`** — the instant the first lease on the chunk was minted, the first `LeaseFact.minted_at` in
  `ChunkFacts.leases` (`hub/domain/chunk/model.py:337`). A restart or a requeue later mints further leases and does not
  move it. `ChunkFacts` gains `started_at()` beside `completed_at()` (`:1404`), since nothing derives it today. A run
  stopped before any runner took it up has no start; for it the anchor reads `scheduled`.
- **`finished`** — `ChunkFacts.completed_at()`.

Because a last run must be terminal before its errand can be due at all, `finished` always has its instant when it is
needed.

## The spacing

With a spacing and a last run, the errand is **eligible** from the anchor's instant plus the spacing. With no last run,
the spacing has nothing to count from, and the errand is eligible at once.

## The schedule

A schedule's **moments** are the instants its cron expression names, read as wall-clock time in its `zone`, exactly as
Debian's cron reads them. A moment a daylight-saving change skips fires at once when the clock jumps past it, and a
moment the change repeats fires once, never twice. The day-of-month and day-of-week fields, when both are restricted,
match a day that satisfies either, as cron always has; `0 20 1 * mon` is the first of the month and every Monday. The
operator's page says so, and the board shows a cadence's next moments so that a surprise shows before it fires.

The rule reads only one moment: the **latest moment at or before `now`**, the first the cron library yields iterating
backward from `now`. However long the hub was away, every moment it missed folds into that one, which is what gives the
plan its *no catch-up*.

## The due rule

Given an errand whose last run, if it has one, is terminal:

- **A spacing alone.** Due once `now` is at or past eligibility.
- **A schedule alone.** Due when the latest moment at or before `now` falls after the errand's **reference**, the minted
  instant of its last run.
- **Both.** Due when the latest moment at or before `now` falls after the reference and at or past eligibility. A
  ten-day sweep therefore arrives at the first evening moment once its ten days have passed, and an hour's queue delay
  never drifts it around the clock.

A spacing counts forward from the last run, so a scheduled anchor whose spacing passed while the last run was still live
is due the moment that run settles. One run is minted then, and never a second one for the moments it waited through.

## An errand that has never run

An errand with no run is eligible at once, and so with a spacing alone it is due on the first look. A schedule needs a
reference to fall after, and an errand that has never run has none of its own. It borrows its chore's: the minted
instant of the newest run any errand of the same chore carries, or, when the chore has never minted at all, the instant
the chore was created. A scope linked into an `all` chore on Tuesday morning is therefore swept that evening with its
siblings, not at once for a moment that fell before it joined. A new chore waits for its first moment rather than firing
for the last one before it existed.

## Checked every minute, to the minute

The cadence sweep runs every 60 seconds ([mint.md](./mint.md) §The sweep). Its timer waits after each pass and is
jittered once at boot (`hub/app.py:223`), so its ticks align with no minute boundary, and the rule never asks whether
`now` is a moment, only whether a moment has fallen. A schedule therefore fires within a minute of the moment it names,
plus the length of a pass. A hub that was down at eight finds the errand due on its first pass back, and mints it once.

Every one of these timings is proven on the controlled clock, by moving time and stepping the pass, never by waiting
([carry-over.md](./carry-over.md) §Proven on the controlled clock).

## What a read shows as next

A read that shows an errand ([surfaces.md](./surfaces.md) §Errands) also shows when it comes **due next**, where that
can be known: with a live last run it cannot, and the view says the errand is waiting on that run. Otherwise it is the
later of eligibility and, under a schedule, the first moment after the reference, the first the cron library yields
iterating forward from it. It is a forecast for the operator. The minting pass never reads it.
