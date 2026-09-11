# Plan — `epic:throttling`

A runner today takes any work, at any hour, at whatever it costs. The platform is assuming an operator who wants
everything immediately and at any price, and gives them no way to say otherwise. But the machine that does the work is
also the machine that pays for it, and its owner has opinions: a weekly allowance should last the week, a laptop should
be theirs again during working hours, and a daily figure should be a figure and not an aspiration. Spend caps park a
chunk after the money is gone, which is a floor rather than a plan. Pacing decides whether the work starts at all.

## What to build

- **Appetite as the runner's own configuration.** Every lever belongs to the machine, is composable with the others, and
  is separately optional — so one runner paces a weekly allowance, another stops at a daily figure, a third only works
  overnight, and a fourth declares nothing and behaves exactly as it does today.
- **Pacing against the window.** The share of a subscription window already spent, measured against the share of the
  window elapsed: a runner six days into seven with a day's budget left declines to start anything new.
- **A ceiling in dollars per day**, for the operator who would rather think in money than in windows.
- **Hours and days.** The machine states when it is willing to run, so standing down during the working day is a
  declaration rather than a reminder.
- **One gate on the claim.** All of it resolves into a single question asked before a runner claims work, which any
  lever may answer no — and a runner that is declining says which lever said so, on the board and from the CLI.

## Open questions

- Where the subscription window's spend and its boundaries come from, given that the harness's own billing surface is a
  receipt rather than an API.
- Whether a declined claim is worth recording as a fact — useful for explaining a slow night, and noisy at one row per
  refusal.
- How pacing composes with the spend caps that already exist, so an operator who sets both is not surprised by which one
  stops them.
