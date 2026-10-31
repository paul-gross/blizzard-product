# Mock contract

A release is proven by `blizzard` and `blizzard-mock` and nothing else. The mock fleet is what the service tier, the
crash sweep, and the e2e loop run against, so a release names the mock commit it was proven with as exactly as it names
its own. `blizzard-mock` keeps its own `pyproject.toml` version unchanged; its tags and branches are its release
identity, and they carry blizzard's names one for one ([train.md](./train.md) §Names).

## Which mock proved it

Today a `push` run checks the mock out at a resolved ref (`upper-tiers.yml`, *resolve blizzard-mock ref*) and keeps no
record of the commit that ref pointed at. The resolve step records it: it writes the mock's commit sha as a step output,
and the `push` workflow uploads it as a `proven-against` artifact naming the blizzard sha and the mock sha together. A
candidate is cut at the mock commit the `proven-against` artifact of the candidate's own `push` run names, never at the
mock's `master` tip at the time of the cut, which may have moved since.

## Branches and tags

Cutting `release/X.Y` creates it in both repositories, blizzard's at the candidate commit and the mock's at the commit
that proved it. Every tag on blizzard's branch is pushed on the mock's branch too, with the same name, at the mock
branch's tip — on every candidate, stable release, and patch, whether or not the mock changed since the last tag. The
mock's tag is pushed first and blizzard's last, since blizzard's `release` run fails when the mock's tag is missing.

A fix that needs the mock to change is carried to the mock's line the same way a blizzard fix is carried to blizzard's
([patches.md](./patches.md) §Carrying a fix back): on `master` first, then by cherry-pick onto a same-named
`patch/X.Y/<slug>` branch and a pull request into the mock's `release/X.Y`, under the same provenance rule. The mock's
`pr.yml` runs on `release/**` as blizzard's does.

## Resolving the mock's ref

One resolution, written once as a composite action (`.github/actions/resolve-mock-ref/`) that `upper-tiers.yml` and each
of `release.yml`'s three tier jobs call, replacing both today's inline step and the tag run's checkout of the mock's
default branch:

| Run                                       | Mock ref                                                                            | When it is absent |
| ----------------------------------------- | ----------------------------------------------------------------------------------- | ----------------- |
| A `v*` tag                                | the same tag                                                                        | the run fails     |
| A pull request into `release/X.Y`         | the head branch's name on the mock if it exists, else the mock's `release/X.Y`      | the run fails     |
| A pull request into, or push to, `master` | the head branch's name on the mock if it exists, else the mock's `master`, as today | —                 |

Checking out one tag in both repositories, laid out as siblings, therefore reproduces exactly what that release was
proven against.
