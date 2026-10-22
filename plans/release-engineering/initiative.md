---
initiative: release-engineering
refinement: scaffolded
epics:
  - first-release
  - compat-floor
  - notes
  - release-lines
  - dark-launch
  - supply-chain
---

# Plan — `initiative:release-engineering`

Today every merge to `master` is a release in all but name. The hosted hub pulls whatever `master` most recently proved,
the runners beside it are rebuilt from `master` after every landing, and the only version tag blizzard has ever cut is a
release candidate from the summer. That served while changes were small. It stops serving when the work is the size of
multi-tenancy or projects: a capability that lands over weeks in dozens of pieces, every one of which has to leave each
running hub and runner working, because every one of them is deployed the moment it lands. Compatibility judged one
commit at a time is a promise nobody can reason about, and the skew window has already had to grant exceptions case by
case to survive it.

The operator who runs blizzard outside this workspace feels the same thing from the other side. The application
architect with a fleet of their own wants to upgrade when they choose, read what a release will ask of them before they
pull it, and know which of their runners will still work against which hub. A stream of commits answers none of those
questions; a release answers all three.

This initiative separates three events that are one today: **landing**, **releasing**, and **deploying**. `master` is
where work comes together. A release is a deliberate, versioned cut, made on a weekly train, that carries its own notes
and is held to the version rules. Every deployment, blizzard's own included, runs a release.

Much of the machinery already exists in the `blizzard` repo: a version scheme naming every surface a release can break,
the breaking-change marker on commits, the hub↔runner wire check, a working downgrade for every store revision, and a
tag-triggered workflow that publishes a wheel, a multi-arch image, and generated notes. What is missing is anyone
consuming its output, and compatibility measured against what was actually shipped.

## The stance

- **The contract is what a release shipped.** Surface that has only ever lived on `master` is still wet clay: it can be
  renamed, reshaped, or removed freely until a release carries it. Once released, it answers to the version rules. This
  is what lets a large epic land in pieces without each piece pretending to be a public promise.
- **The calendar sets the release, not the scope.** A release leaves on its day carrying whatever is finished, and
  nothing waits for an epic. Scope is what the release notes headline; it is never what the version number counts.
- **The version number says what an upgrade costs.** A patch is a fix and safe to take without reading. A minor adds and
  never takes away. A major is the only release allowed to break anything, and it arrives on a date the operator knew
  about months in advance.
- **Every release operation is one an agent can carry out.** Cutting, promoting, patching, and breaking are each a
  written procedure in the workspace's operator workflows, `workspace:/context/project/workflows/`, so the operator says
  *patch 0.7 with this fix* and an agent carries it from `master` to a published release without improvising a step. A
  release process that lives in one person's memory is the most fragile part of a project that wants to be released by
  its own fleet.
- **Blizzard's own fleet runs every release candidate first.** The dogfood instance stops following `master` and starts
  following releases, as their first and least forgiving user. Feedback arrives a week later than it does today, and
  that is the trade: a bad landing becomes a bad candidate, caught before anyone else pulls it.

## The rhythm

Blizzard keeps the rhythm of a mature open-source project, closest to Angular's and Node's: frequent small releases that
only add, and breaking changes gathered into two predictable dates a year.

| Release   | What it may carry                                                                  | How often                                      |
| --------- | ---------------------------------------------------------------------------------- | ---------------------------------------------- |
| **Major** | the removals its cycle deprecated, and nothing else breaking that was not declared | twice a year, on dates published a cycle ahead |
| **Minor** | new capability — epics landing, flags turned on, deprecations announced            | every week, on the release train               |
| **Patch** | fixes only, backported onto a released line                                        | as a fix needs, with no wait for the train     |

Between majors, `master` only adds. In the weeks before a major, `master` opens for its removals: its candidates carry
the next major's number, while the outgoing major's final line continues from its release branch, so an operator never
loses a fix for choosing to wait.

Blizzard stays on `0.x` until `milestone:hardening` is reached, and the release that reaches it is 1.0; the six-month
major cadence counts from there. Until then, a breaking change waits for a minor after one that marked it deprecated, so
the stricter discipline is already the habit by the time 1.0 makes it a promise.

## Compatibility measured against releases

The compatibility checks ask today whether a change breaks the commit before it, or the commit the `edge` image last
published. Once deployments run releases, both ask the wrong question.

The skew window is first restated in terms the rhythm can keep. With a minor every week, a runner allowed to lag its hub
by one minor is a runner with a week to upgrade, which is no window at all. The window is drawn around majors instead: a
runner works against any hub of its own major at the same minor or later, and a hub of a new major still accepts runners
from the previous major's final minor, so an operator upgrades the hub first and the runners after it, at their own
pace. A hub never requires a runner newer than itself. This is the shape Kubernetes keeps between its control plane and
the agents on each node, and it is the closest relative blizzard's hub and runner have. Before 1.0 there are no majors
to draw it around, so the window has a floor instead: a hub accepts runners of every line from the floor up to its own,
and the floor rises only in the change that removes surface a released minor deprecated. A week's train never becomes a
week's deadline.

The baseline every compatibility check compares against becomes the **supported releases**: the newest release of the
oldest line still inside the window, and the newest release of each major above it. A minor only adds, so the newest
release of a major carries every promise that major's earlier releases made. A change that breaks a commit no release
ever carried breaks nobody and should pass. A change that keeps the previous commit happy but breaks a runner from last
month's release must fail.

The check runs on every pull request, not only at the cut. It is a static diff and costs seconds, and a break found at
the cut is found at the worst moment: long after it landed, tangled up with everything else in the release, and holding
the release hostage. On a pull request it lands on the change that made it, while its author still holds the context.
The release workflow runs it once more as a backstop before anything is published.

| Where                     | Compared against                       | A break passes when…                                                                                                                                                                     |
| ------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pull request to `master`  | every supported release                | `master` is open for a major — before 1.0, the change raises the floor — and the break removes surface a released minor already deprecated, with the breaking marker and an upgrade note |
| Pull request to a release | every release already cut on that line | never — a patch adds or it is not a patch                                                                                                                                                |
| A release tag             | every supported release, as a backup   | the tag is a major — before 1.0, one whose floor rose — and every break it carries is named in its notes                                                                                 |

The same baseline governs every surface the version scheme names, not only the hub↔runner wire: the HTTP API through a
diff of its published OpenAPI document, configuration keys, the trace and egress contracts, and the store schema.

A static diff cannot see behavior, so the checks gain an **upgrade rehearsal**: each supported release's hub, seeded
with real-shaped data, is upgraded to the candidate, its migrations walked forward and back, and that release's runner
then drives a chunk to completion against the new hub. It runs on every push to `master` and on every candidate; a
stable release is its candidate's commit, and stands on its candidate's proof. It is the one proof that the promise the
version scheme makes is a promise the software keeps.

At runtime, the skew window stops being a sentence in a document. A runner names its version when it registers, and a
hub refuses one outside the window with a message that names both versions and the window, rather than failing later in
some unrelated place.

The store follows the same discipline as the wire: **expand, then contract**. A minor only adds — columns, tables,
values — and what it replaces is removed at the next major, once nothing inside the window still reads it.

The store's history contracts on the same rhythm. Every revision ever written is replayed by every fresh install, every
hub a test starts, and every store a test builds, long after the last running store has moved past it; a contributor
replays every schema decision the project has ever made just to get an empty database. Once the oldest supported release
has run a revision, nobody inside the window will ever need to run it again. So at each major, every revision up to the
last one the oldest supported release carries is folded into a single baseline that recreates the schema in one step.
The baseline keeps the identifier of the last revision it absorbs, so a store already at or past that point sees nothing
change, and only a store older than every supported release is turned away, with a message naming the release to step
through first. That is no new demand: the window already asks it of anyone that far behind. Before 1.0, with no majors
on the calendar, the fold rides the first cut of each month instead. Two proofs hold it honest: a store built from the
baseline matches, table for table, one built by replaying the history it replaced, on every database blizzard supports;
and the upgrade rehearsal from the oldest supported release still walks forward to the candidate and back again, down as
far as the baseline.

## What to build

Each epic's plan owns its requirements; this is the cut, in the order the epics are meant to land.

- **[`first-release`](./first-release/index.md) — a line in the sand.** The first real release, the skew window restated
  around majors, and the dogfood instance moved off `master`: the hosted hub and the runners each pinned to an exact
  release, moved only by a reviewed change that opens itself when a release publishes. The `edge` image keeps publishing
  for anyone who wants it; nothing of ours follows it.
- **[`compat-floor`](./compat-floor/index.md) — compatibility held against what shipped.** Every surface the version
  scheme names diffed against the supported releases on every pull request and again at the tag, the per-commit
  comparisons retired, the upgrade rehearsal, the version handshake at registration, and expand-then-contract on the
  store.
- **[`notes`](./notes/index.md) — written with the change, not reconstructed after it.** A change an operator would
  notice carries its own note, committed beside the code and reviewed with it, and a release's notes are assembled from
  them — at the fleet's pace, a week of commit subjects is not something a person can read. Deprecations name the major
  that removes them, so a major's upgrade guide gathers itself.
- **[`release-lines`](./release-lines/index.md) — the train and a way to patch.** A candidate every week, run by
  blizzard's own fleet and then promoted; a branch for every release, so a fix reaches a released line by cherry-pick
  from `master`; a deliberately short promise of which lines are patched; `blizzard-mock` mirroring every branch and
  tag; majors on their published dates; and the store's history folded into a baseline.
- **[`dark-launch`](./dark-launch/index.md) — `master` that is always releasable.** Work that spans weeks lands behind a
  flag that is off by default, the release that turns it on is the one whose notes describe it, and a flag hides
  behavior, never contract, so the compatibility checks stay blind to flags.
- **[`supply-chain`](./supply-chain/index.md) — artifacts a stranger can trust.** Images signed, with a bill of
  materials and build provenance attested; checksums beside every attached file; and a runner installable at a named
  version from the release itself. Publishing to a package index waits for the project's rebrand.

## The procedures live in the workspace

None of the epics is finished when its mechanics work. Each is finished when an agent told to perform its operation by
name can find the procedure, follow it, and prove the outcome, in the same way
`workspace:/context/project/workflows/release.md` already carries a tag from a green `master` to a confirmed release. A
release operation is an operator's pass, never a fleet worker's, so its procedure belongs beside that one in the
workspace's operator workflows, written there by the operator by hand, since no fleet worker's environment holds the
workspace. The procedures a fleet worker follows inside a change — judging a break, writing a change note, landing work
behind a flag — stay under `blizzard-context`'s `workflows/`, and the rules every procedure leans on — what a patch may
carry, what a change note must say — under its `standards/` and `verification/`, where every other rule a change is held
to already lives.

The patch is the procedure that most needs writing down, because it is the one run under pressure and rarely enough that
nobody remembers the last time. It starts from the fix on `master`, decides whether the fix can wait for the train,
finds every supported line that carries the bug, carries the fix back by cherry-pick — into `blizzard-mock`'s matching
branch too, when the fix needs the mock to change — adapts it by hand where the line has drifted, writes it on the old
line alone when the code it fixes no longer exists on `master`, proves the branch against every release already cut on
the line, bumps the version on the branch, tags both repositories there, and confirms the published release — and then
moves the dogfood instance onto it.

| Procedure                                                                                  | Lands with      |
| ------------------------------------------------------------------------------------------ | --------------- |
| Cutting a release, and moving the instance's pins onto it or back off it                   | `first-release` |
| Judging whether a change breaks a supported release, and what lets a break through         | `compat-floor`  |
| Writing a change note, an upgrade note, and a deprecation that names the major removing it | `notes`         |
| Cutting the weekly candidate, and promoting it to stable                                   | `release-lines` |
| Patching a released line: fix first on `master`, carry it back, tag from the branch        | `release-lines` |
| Opening `master` for a major, and cutting the major on its date                            | `release-lines` |
| Folding the store's migration history into a baseline at the oldest supported release      | `release-lines` |
| Landing work behind a flag, turning it on in a release, and retiring it                    | `dark-launch`   |
| Confirming a release's signatures, attestations, and published wheel                       | `supply-chain`  |

The workspace's own runbook for the dogfood instance follows the same move, from redeploying after every landing to
redeploying onto each release, and defers to these procedures rather than restating them.

## What this is not

This is not `epic:advanced-deployment`, which gives an operator's own projects a declared way to deploy what they land.
This initiative is about how blizzard itself is released. The two meet at the dogfood instance: once declared deployment
exists, the instance's upgrades can travel it, deploying releases rather than landings.
