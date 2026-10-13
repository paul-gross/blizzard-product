---
epic: release-engineering
refinement: scaffolded
slices:
  - name: first-release
    status: horizon
  - name: compat-floor
    status: horizon
  - name: notes
    status: horizon
  - name: release-lines
    status: horizon
  - name: dark-launch
    status: horizon
  - name: supply-chain
    status: horizon
---

# Plan — `epic:release-engineering`

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

This epic separates three events that are one today: **landing**, **releasing**, and **deploying**. `master` is where
work comes together. A release is a deliberate, versioned cut, made on a weekly train, that carries its own notes and is
held to the version rules. Every deployment, blizzard's own included, runs a release.

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
- **Blizzard's own fleet runs every release candidate first.** The dogfood instance stops following `master` and starts
  following releases, as their first and least forgiving user. Feedback arrives a week later than it does today, and
  that is the trade: a bad landing becomes a bad candidate, caught before anyone else pulls it.

## The rhythm

Blizzard keeps the rhythm of a mature open-source project, closest to Angular's and Node's: frequent small releases that
only add, and breaking changes gathered into two predictable dates a year.

| Release   | What it may carry                                                                  | How often                                      |
| --------- | ---------------------------------------------------------------------------------- | ---------------------------------------------- |
| **Major** | the removals its cycle deprecated, and nothing else breaking that was not declared | twice a year, on dates published a cycle ahead |
| **Minor** | new capability — slices landing, flags turned on, deprecations announced           | every week, on the release train               |
| **Patch** | fixes only, backported onto a released line                                        | as a fix needs, with no wait for the train     |

Between majors, `master` only adds. In the weeks before a major, `master` opens for its removals: its candidates carry
the next major's number, while the outgoing major's final line continues from its release branch, so an operator never
loses a fix for choosing to wait.

Blizzard stays on `0.x` only until a run of weekly releases has proven the process, and then declares 1.0; the six-month
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
the agents on each node, and it is the closest relative blizzard's hub and runner have.

The baseline every compatibility check compares against becomes the **supported releases**: the newest release of each
major still inside the window. A minor only adds, so the newest release of a major carries every promise that major's
earlier releases made. A change that breaks a commit no release ever carried breaks nobody and should pass. A change
that keeps the previous commit happy but breaks a runner from last month's release must fail.

The check runs on every pull request, not only at the cut. It is a static diff and costs seconds, and a break found at
the cut is found at the worst moment: long after it landed, tangled up with everything else in the release, and holding
the release hostage. On a pull request it lands on the change that made it, while its author still holds the context.
The release workflow runs it once more as a backstop before anything is published.

| Where                     | Compared against                     | A break passes when…                                                                                                                          |
| ------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Pull request to `master`  | every supported release              | `master` is open for a major, and the break removes surface a released minor already deprecated, with the breaking marker and an upgrade note |
| Pull request to a release | that line's first release            | never — a patch adds or it is not a patch                                                                                                     |
| A release tag             | every supported release, as a backup | the tag is a major, and every break it carries is named in its notes                                                                          |

The same baseline governs every surface the version scheme names, not only the hub↔runner wire: the HTTP API through a
diff of its published OpenAPI document, configuration keys, the trace and egress contracts, and the store schema.

A static diff cannot see behavior, so the checks gain an **upgrade rehearsal**: the previous release's hub, seeded with
real-shaped data, is upgraded to the candidate, its migrations walked forward and back, and the previous release's
runner then drives a chunk to completion against the new hub. It runs on every push to `master` and on every tag. It is
the one proof that the promise the version scheme makes is a promise the software keeps.

At runtime, the skew window stops being a sentence in a document. A runner names its version when it registers, and a
hub refuses one outside the window with a message that names both versions and the window, rather than failing later in
some unrelated place.

The store follows the same discipline as the wire: **expand, then contract**. A minor only adds — columns, tables,
values — and what it replaces is removed at the next major, once nothing inside the window still reads it.

## What to build

- **`first-release` — a line in the sand.** Cut the first real release from `master`, give each boundary in the upgrade
  guide the version it belongs to, restate the skew window around majors, and move the dogfood instance onto releases:
  the hosted hub pinned to a release tag by a reviewed change in its deployment repository, the runners installed from
  the release's own wheel, and the runner redeploy owed after a release rather than after every landing. The `edge`
  image keeps publishing for anyone who wants it; nothing of ours follows it.
- **`compat-floor` — compatibility held against what shipped.** Every compatibility check compared against the supported
  releases, on every pull request and again at the tag; the checks that compare against the previous commit or against
  `edge` retired; the OpenAPI, configuration, and schema diffs alongside the wire's; the upgrade rehearsal; the version
  handshake at registration; and expand-then-contract enforced on the store's migrations.
- **`notes` — written with the change, not reconstructed after it.** A change an operator would notice carries a short
  note of its own, committed beside the code and reviewed with it, and a breaking one carries its upgrade note the same
  way. The release's notes are assembled from those, with the commit log as a fallback rather than the source — at the
  fleet's pace, a week of commit subjects is not something a person can read. Each minor lists what it deprecates and
  the major that will remove it, and a major's notes gather every removal its cycle announced into one upgrade guide.
- **`release-lines` — the train and a way to patch.** A minor cut every week: a release candidate first, run by
  blizzard's own fleet for the week, then promoted to stable as the next candidate is cut. Each release gets a branch at
  its candidate, so a fix reaches a released line as a patch without dragging along everything `master` has learned
  since; the majors are cut on their published dates from a `master` opened for them. Once `epic:cadence` lands, cutting
  the candidate is a routine the fleet runs for itself.
- **`dark-launch` — `master` that is always releasable.** Work that spans weeks lands behind a flag that is off by
  default, so an unfinished capability can share `master` with a weekly cut. The release that turns a flag on is the
  release whose notes describe the capability, and the flag is retired once it has been on for a release.
- **`supply-chain` — artifacts a stranger can trust.** Images signed, with a software bill of materials and build
  provenance attested; checksums beside every attached file; the wheel published to the Python package index; and a
  runner installable at a named version, so an operator pins what they run as easily as they pin the hub.

## What this is not

This is not `epic:advanced-deployment`, which gives an operator's own projects a declared way to deploy what they land.
This epic is about how blizzard itself is released. The two meet at the dogfood instance: once declared deployment
exists, the instance's upgrades can travel it, deploying releases rather than landings.

## Open questions

- How long a major's final line keeps receiving fixes once the next major ships, and whether fixes that matter to an
  operator's security are held to a longer promise.
- How many weekly releases count as the process proven, and so when 1.0 is declared.
- Whether the hosted hub follows its major's moving tag, taking each week's minor unattended, or is pinned to an exact
  version and moved only by a reviewed change.
- Whether surface that ships in a release behind a flag that is off counts as released for the compatibility checks.
- How a release records the commits of the repositories that proved it — the mock fleet and the workspace its upper
  tiers check out — so that the proof behind a release can be reproduced.
