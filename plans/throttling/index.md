---
epic: throttling
refinement: scaffolded
status: horizon
---

# Plan — `epic:throttling`

A runner today takes any work, at any hour, at whatever it costs. The platform is assuming an operator who wants
everything immediately and at any price, and gives them no way to say otherwise. But the machine that does the work is
also the machine that pays for it, and its owner has opinions: a weekly allowance should last the week, a laptop should
be theirs again during working hours, and a daily figure should be a figure and not an aspiration. Spend caps park a
chunk after the money is gone, which is a floor rather than a plan. Pacing decides whether the work starts at all.

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
  runs out.
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
  than a sample taken minutes ago. It marks the subscription exhausted until the reset the harness named, outranking
  whatever the last sample said, and asks for a fresh sample rather than waiting out the cadence. Without it, a gate
  trusting a five-minute-old 94% would claim, spawn, and hit the wall again on every tick until the sampler caught up.
- **The line holds mid-graph.** A claim is only the first of many starts: a chunk at a plan step with 90% of the window
  spent, against a line drawn at 95%, must not go on to the build step that carries it to 96% merely because it was
  claimed in time. Every lever is therefore asked again before each step of a chunk already held, and a chunk the gate
  declines parks between steps, its claim, session, and retries intact, until the gate opens. The lease that met an
  outright usage limit mid-step parks in place the same way.
- **Holding or letting go.** Whether a chunk parked mid-graph waits for its own runner's gate to reopen, or is released
  to a runner whose subscription still has room, is one more setting in the runner's throttling configuration. Holding
  is the default: the chunk keeps its place and its warm session. Releasing keeps the work moving at the price of a cold
  session on the other side, and an operator can turn it on for one runner to see how it behaves without committing the
  fleet to it. A release is the same move `epic:resilience` makes when it falls back to another route before waiting,
  and the two share it rather than each building their own. A second setting makes the same choice for a lease the usage
  limit stops mid-step, and there letting go costs more: the half-done step goes with the warm session, and the runner
  that takes the chunk starts that step again from its beginning. Holding is the default there too.
- **It clears the way it arrived.** Once the reset has passed and a fresh sample shows the window open again, the
  subscription is green: its runners claim again and its parked leases resume on their own, with nobody at the keyboard.
  A usage limit is a throttle and never a breaker: it trips nothing a person must clear, because exhaustion is a fact
  that expires.
- **One gate before every start.** All of it resolves into a single question asked before a runner claims a chunk or
  starts its next step, which any lever may answer no — and a runner that is declining says which lever said so, on the
  board and from the CLI, naming the subscription and its expected reset when that is the reason. Declining leaves no
  fact behind: a row per refusal would be noise rather than record. A pair marking when throttling engages and lifts may
  earn its place later.
- **The settings in plain view.** An operator who wrote a pacing line into a file weeks ago should not have to open that
  file to learn why the machine is idle tonight. Wherever the runner itself shows its details — `blizzard runner status`
  and the runner's own panel — it shows every lever it is configured with, each with its value and whether it says yes
  or no right now, and beside them the subscription's floor, which is always there whether or not anything was
  configured. The settings stay on the machine that owns them: the hub learns which lever is declining, and nothing
  more, so the board is not committed to showing them and registration does not carry them.
