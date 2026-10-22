---
epic: release-engineering/dark-launch
refinement: refined
status: in-progress
---

# Plan — `epic:release-engineering/dark-launch`

Once a release leaves every week carrying whatever is finished, `master` has to be finished every week. It rarely is. An
epic the size of multi-tenancy or the runner host lands over weeks in dozens of pieces, and on the morning of a cut some
of those pieces are a route nobody calls yet, a board screen with half its buttons, a runner step that works for the one
case its author has proven so far. Today the only way to keep that out of a release would be to keep it off `master`:
long-lived branches, rebased for weeks against everything else landing, merged in one frightening piece at the end. That
is the habit a weekly train is meant to replace, not the price of having one.

The operator meets the same problem from the other side. What they upgrade to is what the release notes describe, and a
capability that turns up in their fleet before its notes do — a new verb in the CLI, a new state on the board, a runner
behaving differently overnight — is a surprise they had no way to schedule. A half-built feature that ships switched on
is the worst of both: they cannot use it yet, and they cannot keep it from using them.

This epic lets unfinished work share `master` with a weekly cut. A capability that spans weeks lands behind a flag that
is off by default, so its code is on `master` and in every release while its behavior is in none. The release that turns
the flag on is the release whose notes describe the capability, and the flag is gone once it has been on for a release.
A flag hides behavior, never contract: any route, field, configuration key, or column the work ships is released the day
it ships, flag or no flag, and the compatibility checks hold it to the version rules like any other surface. That keeps
the checks blind to flags, and it means an epic whose shape is still moving keeps that shape off `master` until it
settles.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **One registry of flags, read from one place.** Every flag is declared once, in a registry that travels with the code:
  its name, the epic it belongs to, which part of the fleet it changes — the hub, a runner, or both together — and its
  default. A flag is created off and the registry is the only place its default ever changes, so turning a capability on
  is a reviewed change to one line, made in the release that announces it. Nothing in the registry records dates or
  versions by hand; how long a flag has lived is read from the releases that carried it.
- **Read the same way everywhere.** The hub, each runner, and the board ask the same question — is this flag on — and
  get the same answer. A flag that changes how the hub and its runners talk to each other is the hub's to answer: the
  runner learns it from the identity answer it already reads at the head of every tick, so the two can never disagree
  about it, and a runner older than its hub, inside the skew window, keeps the behavior it was released with. The board
  ships with the hub that serves it and reads the hub's answer; the runner's panel reads its host's.
- **An override for those who build it, and nothing more.** A feature environment, a test, or a fleet chosen to try the
  work early can turn a flag on through a `[flags]` table in its own configuration file. An override is an operator's
  experiment, not a setting: it carries no support promise, it is never per tenant and never on the admin screens, and a
  hub or runner running one says so wherever its version is shown. The capability's real switch is the default.
- **Surface released, behavior hidden.** What a flag hides is what the software does, never what it promises. A route
  behind an off flag is registered and documented and answers with a refusal that names the flag; a field is present and
  empty; a configuration key is accepted, validated, and ignored with a warning; a column is migrated whether or not
  anything writes it. The compatibility checks of `epic:release-engineering/compat-floor` see every one of them as
  released, and do not know flags exist.
- **A flag's life held mechanically.** A check reads the registry at every release that carried it and knows when each
  flag was born, when it turned on, and when it is due to go. It warns on a pull request once a flag has shipped on in a
  stable release, and fails any cut after a later stable release has shipped with it still live; it warns when a flag
  has been off long enough that its work has stalled. Retiring a flag deletes the path that was off, leaving a tombstone
  so an older runner and a stale configuration file still meet a defined answer.
- **The notes learn it from the flip.** The change that turns a flag on is the change whose note describes the
  capability, held to the note rules of `epic:release-engineering/notes`, and the release's notes list every flag the
  release turned on from the registry's difference against the release before.
- **A procedure an agent can follow.** Landing work behind a flag, turning it on in a release, and retiring it are
  written down in `blizzard-context` beside the other procedures a fleet worker follows, with the rule that a flag hides
  behavior and never contract under its standards, so an agent told to land the next piece of an epic dark carries it
  there without improvising the shape.

## Left for later and out of scope

A flag here is a release mechanism, not a product setting. Turning a capability on for one tenant and not another,
rolling it out to a fraction of the fleet, or letting an operator flip it from the admin screens while the hub runs are
all an operator's configuration, and belong to `epic:live-config`'s records the day someone needs them, as the
warn-or-enforce modes already in `blizzard-hub.toml` do today. An experiment that compares two ways of working by their
outcomes belongs to `initiative:lab`. A flag that outlives a major is a flag nobody finished; there is no long-lived
flag, and no kill switch for released behavior — a released capability that misbehaves is fixed by a patch, which
`epic:release-engineering/release-lines` owns.
