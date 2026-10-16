# Versions contract

Paths are relative to the `blizzard` repository.

## The first release

The first release is `0.2.0`, cut by the cut-commit model `epic:release-engineering/release-lines` owns
([train](../../release-lines/spec/train.md) §Versions on `master` and on a branch, §The cut commit). This section states
only what is particular to the first cut.

1. **The cut commit.** A commit on `master` sets `pyproject.toml` to `version = "0.2.0"` and carries the
   `Release-Cut: 0.2.0` trailer. The notes `cut` and `blizzard:flag-lifecycle` run in it if their epics have landed by
   then; the first cut does not wait for them.
2. **`v0.2.0-rc.1`**, tagged at the cut commit once its `push.yml` run is green. The instance moves onto it
   ([carry-over.md](./carry-over.md)) and runs it while the fleet does its ordinary work.
3. **`v0.2.0`**, at the commit of the last candidate the instance ran without a fault, once the operator declares the
   candidate good.

A fault found in the candidate is repaired as release-lines' further candidate, `v0.2.0-rc.2` on `release/0.2`, never by
moving a tag. If the first cut precedes release-lines' procedures, `release/0.2` is created at the cut commit after the
fact, as [its carry-over](../../release-lines/spec/carry-over.md) provides, before any further candidate is cut.

`v0.1.0-rc.1` stays where it is, as the history of a candidate that was never promoted. No `0.1.0` is ever cut.

The stable tag is rebuilt by `release.yml` from the candidate's commit: the same source and the same pinned dependencies
(`uv.lock`, `web/package-lock.json`), a fresh image and wheel. The candidate's own artifacts stay published under the
candidate's version. How promotion works, and whether it reuses the candidate's bytes, is
`epic:release-engineering/release-lines`'.

## What the first cut needs of the version tooling

The cut-commit model needs two of release-lines' changes before `v0.2.0-rc.1` can be tagged, and they land ahead of the
first cut ([carry-over.md](./carry-over.md)): `scripts/check-version-tag.sh` comparing the tag with any `-rc.K` suffix
removed against `pyproject.toml`, and `push.yml`'s dev version naming the next minor, so every `master` build after the
cut commit is `0.3.0.dev<run>+<sha>` and sorts after `0.2.0` and all its candidates. Both are specified by release-lines
([train](../../release-lines/spec/train.md) §Versions on `master` and on a branch); this epic adds no rule of its own to
either.

The wheel's version is the tag's (`BLIZZARD_VERSION`, set by `release.yml` from `GITHUB_REF_NAME`), so the candidate's
wheel and image report `0.2.0rc1`, the PEP 440 normal form of `0.2.0-rc.1`, and the stable release's report `0.2.0`.
Every comparison this epic makes of a reported version against a pin compares normal forms.

## `docs/versioning.md`

`docs/versioning.md` stays the sole owner of the version scheme, what counts as breaking, and the skew window. It
changes in four places.

- **The opening** keeps semver with `MAJOR` at `0` until 1.0 and adds the boundary of every promise: the version rules
  bind what a release shipped. A build no release carried — an `edge` or `sha-` image, a dev wheel, a wheel built from a
  source checkout — carries no promise, and `0.2.0` is the first release that carries one. It points to the initiative's
  rhythm only through the documents that implement it (`epic:release-engineering/release-lines` writes the cadence into
  this file when it lands).
- **§The hub↔runner skew window** is restated around releases, as a policy with a floor. A hub accepts runners of any
  line from the window's `floor` up to its own, at any patch, and never requires a runner newer than itself; a hub is
  upgraded first and its runners after it. Before 1.0, `floor` rises only in the change that removes surface a released
  minor deprecated, to the line that deprecated it, so the window spans every minor since the last removal however often
  the train cuts one. From 1.0 the window is the initiative's: a runner works against any hub of its own major at the
  same minor or later, and a hub of a new major also accepts runners at the previous major's final minor, which is then
  the `floor`. The section states the policy and names where the floor is declared; the declaration (`window.toml`), the
  checks held to it, and the refusal at registration are `epic:release-engineering/compat-floor`'s
  ([baseline](../../compat-floor/spec/baseline.md), [handshake](../../compat-floor/spec/handshake.md)), and this section
  restates none of their mechanics. The paragraph on what is checked keeps describing `blizzard:wire-compat` until
  compat-floor's check replaces it.
- **The two exceptions** — the subscription-usage boundary and the runner-identity boundary — leave this file. Both sit
  between builds no release carried, so neither is an exception to any promise; their operator-facing ordering already
  lives in `docs/upgrade.md`, under the release that crosses them (§`docs/upgrade.md` below). The tenants paragraph,
  which takes no exception, goes with them. The paragraphs on `TurnSegmentView.kind`, the capability-matched peek, and
  the registration's `subscriptions` field stay: they state rules about the wire, not history.
- **§What a tag publishes** keeps its table and adds the rule the instance depends on: a deployment that wants a release
  pins its exact version, and its digest where the deployment can hold one, never `latest` or an `X.Y` line, which by
  design never carry a candidate.

## `docs/upgrade.md`

`## Before you pull` stays, naming the release's **Upgrade notes** as what to read first. A new `## Upgrading to 0.2.0`
holds every boundary section the file has today — subscription usage, file-configured work sources, work-source
identity, capability reporting, runner identity, tenants, host graduation — in the order an operator arriving from
`v0.1.0-rc.1` crosses them, each heading unchanged so existing anchors resolve. It opens with the one fact that governs
all of them: `0.2.0` is reached from `v0.1.0-rc.1` or any `edge` build by stepping through no intermediate release,
since none exists, so every boundary's own ordering applies at once — runners first across the subscription-usage
boundary, the hub first across runner identity, as each section already says. Later releases' sections are added by the
notes each release gathers (`epic:release-engineering/notes`).

`## Pull and recreate` uses the published tag shape: `ghcr.io/paul-gross/blizzard-hub:0.2.0`, with no `v`
(`scripts/image-tags.sh` strips it). `docs/rollback.md` §2 and §4 correct the same shape (`:0.2.0`, not `:v0.1.0`), and
`docs/install.md` is checked for it. The release-notes lookup keeps the `v`, which is the Git tag's:
`gh release view v0.2.0`.

## What does not change

`scripts/image-tags.sh`'s fan-out (exact; exact, `X.Y`, and `latest` for a stable release), `release.yml`'s order of
image before Release, its jobs, and `push.yml`'s `edge` and `sha-<full-git-sha>` publish all stay as they are. The
release notes keep `scripts/release-notes.sh` until `epic:release-engineering/notes` replaces its source; for `v0.2.0`
the notes range is `v0.1.0-rc.1..v0.2.0` (`git describe` on the tag's parent), and its **Upgrade notes** placeholder is
replaced by hand with a pointer to `docs/upgrade.md` §Upgrading to 0.2.0, as `bzh:release` already prescribes.
