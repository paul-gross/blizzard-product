# Plan — `epic:pwa`

The fleet's most time-sensitive moments happen when [`persona:product-owner`](../charter/personas/product-owner.md) is
nowhere near a browser. A chunk that stopped at nine in the evening waits until someone next walks past a terminal, and
the board — the one surface that would tell them — lives on a laptop that is closed. The cheapest honest answer is to
put the board on the home screen: a progressive-web-app shell, laid out for a thumb, with questions arriving as
notifications.

This is one of two answers to the same need, weighed against `epic:android` and `epic:ios`. Which one blizzard invests
in is a deliberately deferred choice.

## What to build

- **An installable shell.** The board installs to a home screen and launches like an application, without a second
  frontend to maintain.
- **A layout for a thumb**, for the few things that matter on a phone: how the night is going, what is waiting on a
  person, and one chunk in detail.
- **Question notifications through the web.** Riding the notification fan-out seam `epic:chat` builds, so this epic adds
  a channel rather than a mechanism.
- **Honesty when stale.** A phone is offline often; a board that is showing an old picture must say so rather than let
  an operator act on it.

## Open questions

- What web push can actually promise on iOS, which is the constraint that decides whether this is sufficient or merely
  cheap.
- Whether the phone layout is the board's own responsive behavior or a route of its own, and which of those the board's
  code wants.
- Whether answering a question from the shell is in scope here or stays with the channels that already do it well.
