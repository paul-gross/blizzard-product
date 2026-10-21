---
epic: throttling
refinement: refined
status: horizon
---

# Plan — `epic:throttling`

A runner today takes any work, at any hour, at whatever it costs. The platform is assuming an operator who wants
everything immediately and at any price, and gives them no way to say otherwise. But the machine that does the work is
also the machine that pays for it, and its owner —
[`persona:application-architect`](../../charter/personas/application-architect.md), who queues work and walks away — has
opinions: a weekly allowance should last the week, a laptop should be theirs again during working hours, and a daily
figure should be a figure and not an aspiration. Spend caps park a chunk after the money is gone, which is a floor
rather than a plan. Pacing decides whether the work starts at all.

Then there is the wall itself. A subscription that runs dry mid-turn stops its harness cold, and every runner spending
that plan stands idle behind a pause only the operator can lift — still standing at breakfast, hours after the window
reset and the allowance came back. The limit was never the runner's: it is a fact about the subscription, with its
expiry printed on it, and it ought to clear the way it arrived, without waking anyone.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **Throttles and breakers.** Every lever this epic adds is a throttle: a temporary pause, under which a runner waits
  and then carries on by itself once the condition holding it clears. A breaker is the other kind of stop — it trips,
  and a person must come and clear it. The breakers that exist today keep their place: the operator's own pause, the
  runner's dollar ceiling over its rolling window, and the per-chunk spend cap. A tripped breaker stops a runner
  whatever its throttles say; the throttles govern only a runner no breaker holds.
- **Appetite as the runner's own configuration.** Every lever belongs to the machine, is composable with the others, and
  is separately optional — so one runner paces a weekly allowance, another stops at a daily figure, a third only works
  overnight, and a fourth declares nothing and is held back only by its subscription's floor, below.
- **Pacing against the window.** The share of a subscription window already spent, measured against the share of the
  window elapsed: a runner five days into a seven-day window with only a day's allowance left is running ahead of its
  week, and declines to start anything new. The reading is the host's own sampling of each subscription's usage windows
  — the same utilization and reset times the board's subscription rows already show — so a pacing lever, or a plain
  threshold such as "only while under 95%", judges by figures the operator can see. The lead a pace allows may narrow as
  the window fills, so a runner that may run a little ahead early in the week is held close to the line as the allowance
  runs out. The reading is the subscription's whole usage, not the fleet's share of it: an owner who spends the same
  plan by hand during the day leaves the runner that much less of the week, which is the point — the allowance being
  paced is theirs.
- **A budget in dollars.** A daily figure is a budget, and a budget throttles: a runner that has spent its day's
  allowance waits for the day to make room, then carries on. It stands beside the dollar ceiling rather than in its
  place. The budget is the line an operator plans their spending against; the ceiling, set above it, is the alarm for
  the night something has gone badly wrong — a runner that spends through its budget and keeps going trips it, and stays
  stopped until a person looks. A runner on a subscription paces in dollars as readily as one on API keys.
- **One way of counting dollars.** Every dollar line, throttle or breaker, counts what was billed plus what was
  estimated — the same figure the board shows for the work. An OpenCode runner on a ChatGPT subscription is billed
  nothing and estimated a great deal, and its budget, its ceiling, and its chunk cap all see that estimate: it
  throttles, and it can trip. The two layers never disagree about what was spent, only about what happens at the line.
- **Hours and days.** The machine states when it is willing to run, so standing down during the working day is a
  declaration rather than a reminder.
- **The subscription's own floor.** Every runner spending a subscription declines new work while that subscription is
  exhausted, whether or not it declares a single lever — the one condition no operator should have to remember to ask
  for. The reading belongs to the subscription rather than to any runner, so every runner spending that plan declines
  together, and none of them is paused: each is simply waiting, and says on whose account. The gate takes a runner's
  declaration at its word: an operator who names a subscription for a runner has signed that runner's harnesses into
  that plan, and the sample read for it is the usage they spend. A reading is the host's own: one account spent from two
  machines is two subscriptions to the gate, and a limit met on one holds back nothing on the other.
- **The harness's word over the sample.** A harness reporting its usage limit is the freshest reading there is, fresher
  than a sample taken minutes ago. It marks the subscription exhausted, expecting it back at the reset the harness
  named, outranking whatever the last sample said, and asks for a fresh sample rather than waiting out the cadence. A
  sample that has not seen the window turn over cannot overrule it: a limit met at what the provider still reports as
  99% stays met until the provider shows a new window, however long its counter lags. Without it, a gate trusting a
  five-minute-old 94% would claim, spawn, and hit the wall again on every tick until the sampler caught up.
- **The line holds mid-graph.** A claim is only the first of many starts: a chunk claimed with 90% of the window spent,
  against a line drawn at 95%, whose plan step carries it to 96%, must not go on to its build step merely because it was
  claimed in time. Every lever is therefore asked again before each step of a chunk already held, and a chunk the gate
  declines parks between steps, its claim, session, and retries intact, until the gate opens. The lease that met an
  outright usage limit mid-step parks in place the same way. The line is drawn between steps, never inside one: nothing
  can know what a step will cost before it runs, so a step admitted just under a line runs to its end and may overshoot
  by its own cost. What catches the overshoot is already there — the floor when a step runs the window dry, the ceiling
  when one runs through the budget.
- **Holding or letting go.** Whether a chunk parked mid-graph waits for its own runner's gate to reopen, or is released,
  is one more setting in the runner's throttling configuration. Holding is the default: the chunk keeps its place and
  its warm session. Releasing puts the chunk back in the queue for whichever runner claims it next — a neighbour whose
  gate is open, or this same runner once its own reopens — and the warm session and the environment it had built are the
  price, paid knowingly. An operator can turn it on for one runner to see how it behaves without committing the fleet to
  it. A second setting makes the same choice for a lease the usage limit stops mid-step, and there letting go costs
  more: the half-done step goes too, and whoever claims the chunk starts that step again from its beginning. Holding is
  the default there as well. This is the fleet's first use of a mid-graph release it means to rely on more broadly, as
  the way an agent's work changes hands without anyone stepping in: `epic:resilience` makes the same move when it falls
  back to another route before waiting, and every later caller shares the one release rather than building its own.
- **It clears the way it arrived.** As soon as a fresh sample shows the window open again — at its reset, or sooner,
  when the owner has bought the window back early — the subscription is green: its runners claim again and its parked
  leases resume on their own, with nobody at the keyboard. A usage limit is a throttle and never a breaker: it trips
  nothing a person must clear, because exhaustion is a fact that expires.
- **When no sample can come.** The floor clears on evidence, and evidence can fail to arrive: a sampler whose
  credentials have lapsed, or a provider whose usage endpoint is down, would hold every runner on the plan for as long
  as the outage lasted. A hold an hour past its expected reset with no sample since clears on the clock, and the next
  start is the probe — a limit met again marks again. A sample that does arrive and shows the window still shut keeps it
  held: the hour answers a sampler that cannot see, never one that sees the window closed.
- **And a hand on it.** An operator who knows better than the reading — a reset redeemed that the provider's counter has
  not caught up with, a sampler reading the wrong account — clears a subscription's floor from the CLI, and every runner
  spending it may start again at once. The clear sets aside the holds standing when it is made and nothing after them:
  the harness meeting the limit again holds the plan again, as it would have without the clear. The floor stays a
  throttle all the same — a person may lift it, and never has to.
- **One gate before every start.** All of it resolves into a single question asked before a runner claims a chunk or
  starts its next step, which any lever may answer no — and a runner that is declining says which lever said so, on the
  board and from the CLI, naming the subscription and its expected reset when that is the reason. Declining leaves no
  fact behind: a row per refusal would be noise rather than record. A pair marking when throttling engages and lifts may
  earn its place later.
- **The settings in plain view.** An operator who wrote a pacing line into a file weeks ago should not have to open that
  file to learn why the machine is idle tonight, and
  [`persona:harness-engineer`](../../charter/personas/harness-engineer.md), moving a lead a point at a time, reads each
  lever's answer beside its setting rather than inferring it from an idle machine. Wherever the runner itself shows its
  details — `blizzard runner status` and the runner's own panel — it shows every lever it is configured with, each with
  its value and whether it says yes or no right now, and beside them the subscription's floor, which is always there
  whether or not anything was configured. The settings stay on the machine that owns them: the hub learns which lever is
  declining, and nothing more, so the board is not committed to showing them and registration does not carry them.

## What this epic is not

**Not an order.** `epic:urgency` decides which chunk a runner reaches for; this epic decides whether it reaches at all.
An urgent chunk waits behind a declining gate like any other, because the gate judges the machine's appetite and not the
work. An operator who needs it now lifts the lever, or lets a runner whose gate is open take it.

**Not a calendar for work.** `epic:cadence` mints a routine's work at the hub when its interval comes due; the schedule
here decides only when one machine is willing to start work. A routine minted on Sunday at four waits in the queue for a
runner whose schedule admits it, as any chunk does.

**Not a fleet budget.** Every lever is one runner's. Nothing here sums a tenant's spend, shares an allowance across
runners, or throttles a project or a chunk rather than a machine. Nor is a subscription one thing across hosts: an
account spent from two machines is two subscriptions to the gate, and a limit one host meets holds back the other only
once that host's own sample shows the window shut.

**Not live.** Every lever is read when the host starts, and an edit takes effect at its next start. Changing a runner's
appetite while the host runs is runtime configuration's to build.

**Not a bound on waiting.** A throttle holds for as long as its condition does. A chunk held behind a pace or a schedule
over a weekend is the configuration doing what it says, so nothing here escalates a chunk for waiting — unlike the
patience `epic:resilience` bounds for an outage nobody chose.

**Not the chunk's own status.** The declining reason is shown on the runner: its status, its panel, the board's runner
row, and the hub's CLI. A chunk held between steps still reads as running on its own card; saying on the chunk which
runner holds it back, and why, is left for later.
