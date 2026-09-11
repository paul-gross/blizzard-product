# Plan — `epic:ios`

The same promise as `epic:android`, on Apple's platform: every hub and runner the operator follows registered in one
place, read as a single view of chunks, questions, and escalations, and reached through notifications that behave like
the platform's own. It enters as an exploration alongside its Android counterpart, and it shares that epic's registry
and aggregation design — whichever ships first decides them. What earns it a separate epic is that the platforms differ
in exactly the places this work lives: the store, the notification plumbing, and what an application is permitted to do
in the background.

## What to build

- **The shared model, bound to iOS.** The hub-and-runner registry and the unified cross-fleet view, implemented against
  Apple's platform rather than redesigned for it.
- **Notifications through APNs**, reliable enough that an operator can stop watching and trust the phone to tell them.
- **Acting from the notification.** Answering a question and unblocking a chunk without a trip to a laptop.
- **Background behavior that survives review.** What the app may do while closed, decided against Apple's rules rather
  than discovered by rejection.

## Open questions

- What App Store review makes of an application whose entire purpose is operating a fleet of coding agents, and what the
  listing has to say to pass.
- How much implementation is genuinely shared with `epic:android` beyond the design, and whether a shared layer pays for
  itself at two platforms.
- Whether iOS's background rules permit the freshness the unified view implies, or whether the app is honest about being
  a notification client with a good list.
