# Plan — `epic:android`

An operator who follows more than one fleet has no single place to see them. `epic:pwa` puts one hub on a home screen,
which is most of what a phone needs — and stops exactly where a person running a work fleet and a personal one begins to
want them side by side. A native application is the shape that can hold that: every hub the operator follows registered
the way a Jira app registers its sites, local runners alongside them, and all of it read as one.

This enters as an exploration, a deliberate maybe. What a native app must earn is the multi-hub view and the
notification reliability a wrapped page cannot give.

## What to build

- **A registry of hubs and runners.** The operator adds each hub they follow and each runner they own, authenticating to
  each separately, and the app remembers them.
- **One unified view.** Chunks, questions, and escalations across every registered fleet, read as a single list rather
  than a set of tabs — which is the whole argument for building this at all.
- **The platform's own notifications.** The fleet's voice arrives the way the phone's other notifications do, reliably
  enough to be trusted while asleep.
- **Acting, not only watching.** Answering a question and unblocking a chunk from the app, since a notification that
  cannot be acted on is only an interruption.

The hub-and-runner registry and the aggregation model are designed once and shared with `epic:ios`; whichever ships
first decides them.

## Open questions

- Whether aggregation across hubs is done on the device or earns a hub-side surface, which is the difference between a
  client and a new piece of platform.
- What parity with the board costs to maintain, and which board surfaces the app deliberately refuses to carry.
- Whether the multi-hub view is genuinely wanted by enough operators to justify a native application, which is the
  exploration's actual question.
