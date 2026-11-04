# When an errand is due

Every rule here is a pure function of the cadence package's model: of the cadence, the errand, the errand's last run,
and one instant `now`, read from the hub's injected clock once per pass (`bzh:injected-clock`). Nothing is stored about
when an errand is next due: the answer is worked out each time it is asked, by the minting pass and by every read that
shows it.

## The errands

A cadence's errands are worked out from its chores each time it is read, never stored. A chore's scopes resolve to its
errands as follows:

- **`all`** — every scope the routine sweeps now: the scopes linked to the routine, less those retired in the project.
  That is the same live set the routine's sweep coverage reads, so a scope linked on Tuesday is an errand on the next
  look, and one unlinked or retired simply leaves the round.
- **A list** — one errand per listed slug, each standing or stopping on its own ([surfaces.md](./surfaces.md) §Needs
  attention).
- **Absent** — one errand over the routine's default scope.

An errand is identified by its chore's id and its scope's id. A scope's slug is unique in its project, so the two never
disagree.

## The last run

A cadence's run is a **work item**, not a chunk. The work item a mint writes — `hub:5943`, say — is the durable thing:
its run identity carries the errand's mark ([mint.md](./mint.md) §The errand's mark), and it keeps that identity
whatever becomes of the chunk that first wrapped it. A chunk is only the work item's current holder, and an unclaimed
one may be grouped away or deleted. So an errand's runs are the work items whose mark names its chore and scope, and its
last run is the newest of them by minted instant, whatever became of it. A run started by hand, or a run of the same
routine minted by another chore, carries no mark of this errand and is none of its runs.

A run's **state** is read through its work item, in this order:

1. **Withdrawn.** A hub item withdrawn is settled. Deleting an unclaimed chunk withdraws the hub item it holds, so a run
   an operator deleted is a run they did not want: it is the errand's last run, settled at the moment it was withdrawn,
   and the errand waits for its next moment or spacing rather than minting it again a minute later.
2. **Delivered.** A hub item delivered is settled, as `done`.
3. **Otherwise, its holder.** The live chunk holding the work item's ref, whichever chunk that now is. Grouping folds a
   chunk into a survivor that absorbs its work refs, so a run grouped away is followed into the survivor, and its state
   is the survivor's. Several runs, of one chore's errands or of many routines, may converge into one chunk this way,
   and each stays its errand's live run until that one chunk settles; none of their errands mints again meanwhile.

The read is one newest-per-key read of the marked work items for every errand of a tenant's cadences
(`bzh:newest-per-key-read`), with each item's closure and its holder's status read in bulk beside it, never a read per
errand. An errand whose last run's holder cannot be resolved at all is treated as running, never as never-run, so a read
that loses a holder can never mint a duplicate.

A last run is **unfinished** while it is neither withdrawn nor delivered and its holder's status is not terminal.
Blizzard has no failed status and no cancelled one, so the plan's three outcomes read as these:

- **Finished** is a holder at `done`, or a delivered item.
- **Cancelled** is a holder at `stopped`, or a withdrawn item. Each is terminal, and the spacing counts from it.
- **Failed** is a holder escalated to `needs_human`. It is not terminal, so the errand waits on it, indefinitely, as it
  waits on any live run, and a failure is never quietly retried. An operator settles it by stopping it, or starts the
  routine again by hand, a run that belongs to no errand.

## One live run

`bzh:errand-one-live-run`: an errand whose last run is unfinished is never due. This is the first test, before any
moment is read, and the mint re-establishes it under its lock ([mint.md](./mint.md) §The mint's unit of work). A fleet
behind on its queue therefore holds one run per errand, and an errand whose run waits on a person waits with it.

## The three anchors

A spacing counts from one moment of the last run, chosen by the cadence's `anchor`:

- **`scheduled`** — the instant the work item was minted.
- **`started`** — the instant the first lease on the run's holder was minted, which a later requeue's lease does not
  move. The chunk's facts gain this reading beside the completion they already derive, since nothing derives a start
  today. A run grouped into a survivor takes the survivor's start, since the survivor is the chunk that ran it. A run
  settled before any runner took it up — stopped, or withdrawn by its chunk's deletion — has no start; for it the anchor
  reads `scheduled`.
- **`finished`** — the instant the holder became terminal, or the item was withdrawn or delivered, whichever came first.

Because a last run must be terminal before its errand can be due at all, `finished` always has its instant when it is
needed.

## The spacing

With a spacing and a last run, the errand is **eligible** from the anchor's instant plus the spacing. With no last run,
the spacing has nothing to count from, and the errand is eligible at once.

## The schedule

A schedule's **moments** are the instants its cron expression names, read as wall-clock time in its `zone`, as Debian's
cron reads them. Two edges follow from that, and the schedule module's tests pin both against the library:

- **A moment the clock skips** fires at the first instant after the jump: `30 2 * * *` fires at 03:00 on the morning the
  clocks spring forward. An expression naming every hour simply has no moment in the hour that never happened.
- **A moment the clock repeats** fires once when the expression names a fixed hour: `30 1 * * *` fires at the first
  01:30 on the night the clocks fall back. An expression naming every hour fires in each real hour, so `0 * * * *` fires
  at both 01:00s, an hour of real time apart, as an hourly rhythm should.

The day-of-month and day-of-week fields, when both are restricted, match a day that satisfies either, as cron always
has: `0 20 1 * mon` is the first of the month and every Monday. The operator's page says so, and the board shows a
cadence's next moments, so a surprise shows before it fires.

The rule reads only one moment: the **latest moment at or before `now`**. The library yields only instants strictly
before the one it iterates back from, so the schedule module starts it just past `now`, and a moment falling exactly on
`now` counts.

## The due rule

Given an errand whose last run, if it has one, is terminal:

- **A spacing alone.** Due once `now` is at or past eligibility.
- **A schedule alone.** Due when the latest moment at or before `now` falls after the errand's **reference**: the minted
  instant of its last run, or for an errand that has never run, the instant it joined (below).
- **Both.** Due when the latest moment at or before `now` falls after the reference and at or past eligibility. A
  ten-day sweep therefore arrives at the first evening moment once its ten days have passed, and an hour's queue delay
  never drifts it around the clock.

A spacing counts forward from the last run, so a scheduled anchor whose spacing passed while the last run was still live
is due the moment that run settles. One run is minted then, and never a second one for the moments it waited through.

## A missed moment fires late, once

The rule never asks whether a moment is happening now, only whether one has fallen since the errand's reference, so a
moment the hub did not see is not lost. It is owed one run, minted on the hub's first pass after it, however late:

- **A restart across the moment.** A hub stopped at 18:59 and back at 19:05 runs its first pass as it boots. The latest
  moment at or before 19:05 is 19:00, which falls after the errand's last run, so the errand is due, and the 19:00 run
  is minted at 19:05.
- **A long outage.** A hub away from Friday to Monday finds the latest moment, Monday's, fallen after the last run, and
  mints one run. Friday's, Saturday's, and Sunday's moments fold into it.

That single run is the plan's *no catch-up*: an errand that has come due is owed one run, however long it has been
overdue, and never one per moment it missed. Its run then becomes the reference, so the moment it was minted for does
not fire it again.

## An errand that has never run

An errand with no run of its own counts from the instant it **joined** its chore: the later of when the chore last set
which scopes it names, and, for an `all` chore, when the scope was linked to the routine. Under a schedule, it is due at
the first moment after it joined, never for a moment that fell before. A scope linked into an evening `all` chore on
Tuesday morning is swept that evening, with a ten-day spacing or without one, and a new chore waits for its first moment
rather than firing for the last one before it existed. With a spacing alone, it is due on the first look, as the plan
decides.

The two instants come from the store. A chore's `scopes_set_at` is its own ([record.md](./record.md) §The tables). The
routine's scope link gains a `linked_at`, stamped from this epic on whenever a scope is linked. A link made before then
has none, and reads as linked before the chore, so the chore's own instant decides.

## Checked every minute, to the minute

The cadence sweep runs every 60 seconds ([mint.md](./mint.md) §The sweep), and once as the hub boots. Its timer waits
after each pass and is jittered once at boot, so its ticks align with no minute boundary, and the rule never asks
whether `now` is a moment, only whether a moment has fallen. A schedule therefore fires within a minute of the moment it
names, plus the length of a pass, and a moment the hub was down for fires on its first pass back.

Every one of these timings is proven on the controlled clock, by moving time and stepping the pass, never by waiting
([carry-over.md](./carry-over.md) §Proven on the controlled clock).

## What a read shows as next

A read that shows an errand ([surfaces.md](./surfaces.md) §Errands) also shows when it comes **due next**, where that
can be known: with a live last run it cannot, and the view says the errand is waiting on that run. Otherwise it is the
later of eligibility and, under a schedule, the first moment after the reference. It is a forecast for the operator. The
minting pass never reads it.
