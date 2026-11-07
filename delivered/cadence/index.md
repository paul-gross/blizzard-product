---
epic: cadence
refinement: pristine
status: delivered
---

# Plan — `epic:cadence`

Three epics already promise the fleet a schedule — mutation testing's corpus pass, ideation's routine, and the trends
that gardening can only draw from repeated measurement — and each of them would otherwise build a timer of its own. The
fleet should keep its own schedule instead. An operator says once what the fleet should do regularly and how often, and
when it comes due the hub mints the work. Eight in the evening, every ten days, the architecture sweep enters the queue
for every scope its routine tends, and each pass waits its turn like any other work.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## Vocabulary

A gardener keeps a calendar of chores — feed the tomatoes every two weeks, water every third evening — and does each one
bed by bed. The same three words carry this epic:

- **Cadence** — the rhythm, and the bucket that keeps it: one record in a project, holding every chore that runs on the
  same schedule and spacing.
- **Chore** — one line in a cadence, configured once: a routine, the scopes it is run over, and how each run is made.
  *Sweep every scope for architecture, in delta mode* is a chore.
- **Errand** — one chore over one scope, the instance a cadence actually mints. A chore over twelve scopes is twelve
  errands, and each errand keeps its own last run and its own clock. Errands are never configured: a cadence works them
  out from its chores each time it looks.

## What to build

- **A cadence is a bucket.** One record, kept in a project, that holds whatever the operator wants done on the same
  rhythm. It does not mirror any one thing: a single cadence can carry several chores over many scopes, and the operator
  reads it as one plain sentence of intent — *every ten days at eight, sweep everything for architecture and tests*. A
  project keeps as many cadences as it has rhythms, and two tenants' cadences never learn of each other.
- **Every chore is a routine.** Every chunk runs a graph, and a routine is what brings one, so a chore carries nothing
  else. What a chore adds is how its routine is run, stated in the same three options a run started by hand already
  takes: its **scopes**, either a fixed list of one or many, or **all**, meaning every scope the routine sweeps when the
  cadence fires, so a scope linked into the routine on Tuesday is swept on the next run without anyone editing the
  cadence, or none, which runs once over the routine's own default scope exactly as a run started by hand does; its
  **mode**, `full` or `delta`; and a **charge note**, the prose appended to each run's charge. Anything a graph can do —
  a sweep today, perhaps a deployment or a release pass tomorrow — a cadence can do by carrying the routine that runs
  it.
- **One routine, many chores.** Because a chore carries its own mode and charge note, the same routine serves many
  rhythms: a weekly delta architecture review and a monthly full one are two chores over one routine. Work that is only
  a sentence — *export last week's runs as a CSV and email it to me* — needs no routine of its own: it rides a
  deliberately plain one, a general-purpose graph with nothing to judge, and the chore's charge note is the whole of the
  work. Four cadences can each carry that plain routine with a different note, and none of them is the other's.
- **Errands, each with a run of its own.** A cadence mints one chunk per errand that is due. Every rule below is judged
  errand by errand, so one scope stuck in the queue holds back only its own next sweep while the other eleven go out on
  time.
- **Linked by slug.** A chore names its routine and its scopes by slug, the way the people who set it up think of them.
  A routine's slug is its lineage, fixed at mint and never reused, so it is as steady a key as any id. Today a routine
  has only a name, doing double duty as that lineage and as what the board shows, so this epic gives it a slug and a
  name as a project already has: the slug its handle, the name free to read well and change. Every routine's slug is its
  name as it stands, so nothing it has recorded moves. Changing what a chore does is a matter of pointing its routine at
  another graph, and every cadence carrying that routine follows without an edit, its lineage and its delta baselines
  intact.
- **Needs attention, never invalid.** A cadence whose chore can no longer run is flagged as needing attention, so the
  operator sees it where they look rather than finding out from a sweep that never came. A retired routine, or a graph
  that no longer resolves, stops every errand of its chore; a scope in a fixed list that is retired, or no longer linked
  to the routine, stops that one errand. A chore over all scopes only ever offers the scopes its routine can still run
  over, so a scope retired or unlinked there simply leaves the round. The cadence goes on minting every errand that can
  still run, and one with nothing left mints nothing at all until someone mends it.
- **A run knows the errand that minted it.** Every chunk a cadence mints records the chore and the scope it was minted
  for, and an errand's runs are only the chunks that carry its own mark. That is what lets four cadences share the plain
  routine, or a weekly and a monthly chore share the architecture routine over the same scope, without any of them
  mistaking another's run for its own. A run an operator starts by hand, and another chore's run of the same routine,
  belong to no errand of this one and hold none of its errands back. The delta baseline is still the routine's own over
  that scope, so the monthly full run sets the baseline the next weekly delta reads from.
- **Two checks: a schedule and a spacing.** A **schedule** names the moments a cadence may fire — four in the afternoon
  every day, or every Sunday at four. A **spacing** names the least time that must pass since an errand's last run
  before it is minted again — seven days, or thirty-six hours. A cadence may carry either or both. With only a schedule,
  every moment it names mints each errand. With only a spacing, an errand is minted the moment its spacing has passed.
  With both, an errand is minted at the first scheduled moment after its spacing has passed, which keeps a ten-day sweep
  arriving in the evening, when the fleet is idle, rather than drifting around the clock with however long the last run
  took.
- **A spacing measured from a chosen anchor.** Every run passes three moments: it is **scheduled** when its chunk is
  created, **started** when a runner takes it up, and **finished** when the chunk is done. A chunk scheduled today may
  start tomorrow and finish the day after, so a spacing says which moment it counts from:
  - **Started**, the default. A sweep reads the code as it stands when it starts, so seven days from one start to the
    next is seven days of change for each pass to read, however long the queue made either of them wait.
  - **Scheduled**, for a rhythm of minting kept regardless of the queue: a run every seven days, counted from when the
    last one entered it.
  - **Finished**, for work that should rest after it is done, as a bed lies fallow: the next run waits out the spacing
    from the moment the last one was.
- **An errand's last run is the last one it minted.** Whatever became of it — finished, failed, or cancelled. A failed
  or cancelled run is not quietly retried: an operator who wants the work again starts the routine by hand, and that run
  belongs to no errand.
- **One live run per errand.** An errand is never minted while its last run is unfinished, as no gardener waters soil
  that is still wet. A fleet behind on its queue does not pile up copies of the same sweep, and a scheduled anchor skips
  forward while an errand's last run still waits rather than stacking another behind it.
- **No catch-up.** An errand that has come due is owed one run, however long it has been overdue. Watering twice because
  yesterday was missed does the plant no good. A fleet switched off for a week mints each overdue errand once when it
  returns, and every scheduled moment it missed folds into that single run.
- **Checked every minute.** The hub looks over its cadences once a minute, and a schedule is stated to the minute, so a
  cadence fires within a minute of the moment it names. A hub that was down at eight finds the errand overdue on its
  first look back, and mints it once.
- **Minting at the hub when due.** The hub creates the work in the cadence's own project, exactly as a run started by
  hand is ingested there, and only a runner serving that project claims it. A fleet behind on work falls behind on its
  own housekeeping rather than displacing an operator's.
- **Promoted, unless the chore says otherwise.** A run started by hand rests behind the promote gate until someone
  admits it, and a sweep minted at three in the morning that waits for someone to promote it has not kept the fleet's
  schedule at all. So a chore promotes what it mints by default, its errands entering the ready queue the moment they
  are minted, and a chore whose work the operator wants to look over first says so and leaves its runs resting behind
  the gate. A cadence is the one thing beside an operator that admits work to the queue, and only because an operator
  configured it to.
- **An edited chore keeps its errands.** Changing a chore's mode, rewriting its charge note, or pointing it at another
  routine is still the same chore, so each of its errands keeps its last run, its spacing, and its one live run. Only a
  chore removed and another added in its place starts afresh.
- **The existing three, moved over.** The epics already promising a cadence declare one here instead of scheduling
  themselves, which is how the machinery earns its place rather than adding a fourth way to do this.

## Left for later

A spacing counts from an errand's own run, and a sweep finishing is not its findings being dealt with. A sweep that
raised a dozen findings on Sunday is finished on Sunday, while the fixes those findings became land on Thursday as work
of their own. A cadence that should wait until last week's findings are resolved needs a condition on open findings, not
on a moment in a run, and that condition is a later addition rather than a fourth anchor.

The release train and the store fold are written as procedures a routine could carry, but today they are an operator's
workflows, carried out where the workspace is, and no fleet chunk's environment holds the workspace. Scheduling them
waits for a routine that can carry them, and a cadence takes that routine up the day it exists, with nothing here
changing. Fact egress keeps the timer it has: its export is a lane of the hub advancing a cursor over facts, not work
minted for a runner, and a cadence only ever mints work.
