---
epic: release-engineering/compat-floor
refinement: refined
status: horizon
---

# Plan — `epic:release-engineering/compat-floor`

Blizzard already checks its own compatibility, and checks it carefully, against the wrong thing. Every pull request is
held to its merge-base and every push to `master` to the commit the `edge` image last published, so the question asked
is always whether this change breaks the commit before it. Once `epic:release-engineering/first-release` moves every
deployment onto releases, nobody runs the commit before. The question then misfires in both directions. It blocks the
author of a large epic from reshaping a route that landed last Tuesday and has never left `master`, a route no runner
anywhere has ever called; and it waves through a break that a `!` acknowledged a month ago, against a runner still
running last month's release on somebody's desk. Neither is what an operator needs held. What they need held is the
promise a release made them.

The operator meets the gap twice. Once in the morning after an upgrade, when a runner that has quietly fallen out of
step with its hub fails somewhere unrelated — a claim that will not parse, a report the hub drops — and the cause has to
be reconstructed from a stack trace. And once on the day they roll a hub back, when a store revision that dropped a
column turns a routine downgrade into lost data.

This epic moves the floor every compatibility check stands on from the previous commit to the **supported releases**:
the newest release on the oldest line the skew window still serves, and the newest release of each major at or above it.
A change is judged against what those releases shipped, on every pull request while its author still holds the context,
and again at the tag as a backstop. A static diff cannot see behavior, so a rehearsal upgrades a real previous release,
data and all, to the candidate, and has the old runner finish a chunk against the new hub. And the window stops being a
sentence in a document: a runner names its version when it registers, and a hub refuses one outside its window with a
message naming both versions and the window, rather than failing later in some unrelated place.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **The supported releases, computed and never chosen.** The skew window is declared once in the tree, as the oldest
  line a hub still serves, and the supported releases follow from it and from the tags: the newest release on that
  oldest line, and the newest release of each major at or above it. Nothing about the baseline is set by hand at check
  time, so the same tree gives the same answer on a laptop, in a pull request, and at the tag.
- **Every surface read from what a release shipped.** Each surface the version scheme names — the hub↔runner wire, the
  HTTP API, the configuration keys and durable paths, the trace and egress contracts, and the store schema — is a
  committed contract document, generated from the code and held to it, so a release's surface is simply that document as
  its tag holds it. Two surfaces gain a document they lack today: the configuration keys each daemon reads, and the
  shape of each store.
- **One check, every surface, against every supported release.** A change that breaks surface no release ever carried
  breaks nobody and passes. A change that keeps the previous commit happy but breaks a runner from last month's release
  fails. A break passes when it removes surface every supported release already marked deprecated, in a change that
  moves the window past the releases that still use it and says so with the breaking marker; after 1.0, only while
  `master` is open for a major, where a break with no deprecation before it also passes once it is declared, with the
  breaking marker and a breaking change note naming it. A patch to a released line passes no break at all.
- **The old floors retired.** The merge-base comparison, the comparison against the last `edge` publish, and the
  first-parent walk that attributes each break to its landing all go, together with the `edge` channel's wait on them.
  `edge` keeps publishing for anyone who wants it, gated by the tiers alone.
- **A rehearsal of the upgrade an operator will actually perform.** On every push to `master` and every candidate tag,
  each supported release's own hub and runner host, installed from its published wheel and seeded with real-shaped data,
  are upgraded hub first to the candidate, its migrations walked forward and back, and the old runner drives a chunk to
  landing against the new hub. It is the one proof that the version scheme's promise is a promise the software keeps.
- **The window held at runtime.** Every runner registration and every host report carries the version it runs. A hub
  refuses a runner outside its window — one too old for it, or newer than it — with a refusal naming the runner's
  version, its own, and the window; the runner halts on it alone and shows it on its panel, and the board shows the hub
  every runner's version and whether it stands inside the window.
- **Expand, then contract, on the store.** A store revision only adds within the window: tables, columns, nullable
  fields, values. What it replaces is dropped only once nothing inside the window still reads it, by the same deliberate
  window-moving change that lets any other break through, so that every downgrade an operator performs inside the window
  is lossless.
- **The procedure for judging a break.** An agent told to judge whether a change breaks a supported release, and what
  would let it through, finds that procedure in `blizzard-context` and the rules it leans on, and follows it to a
  verdict without improvising a step.

## Left for later

The CLI's command tree is already pinned (`contracts/cli/`), but the version scheme does not name it as a surface, so
this epic leaves it out; naming it is a change to the scheme first. Behavioral compatibility beyond what one chunk's
journey exercises is the rehearsal's limit, not a promise this epic makes: it proves the window's common path, and the
upper tiers keep proving the rest at the candidate's own version. Rehearsing against `blizzard-mock` at each release's
own tag waits for `epic:release-engineering/release-lines` to mirror the mock's tags; until then the rehearsal runs the
mock's `master`.

Writing the upgrade note a break owes, and assembling the notes that name it, belong to
`epic:release-engineering/notes`. The window's policy — how far a runner may trail, before and after 1.0 — is restated
by `epic:release-engineering/first-release`; this epic declares it in a form the checks and the hub can read, and
enforces it.
