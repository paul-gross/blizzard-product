# Carry-over

The epic lands into a repository whose releases have been described by their commit logs, whose upgrade guide is a
document of boundaries, and whose `bzh:release` ends with a hand-written edit. None of it is thrown away.

## The order of landings

1. **The format and the assembler.** `changes/` with its `README.md`, `blizzard-change-notes check`, `cut`, and
   `render`, wired into `release.yml` and into the gate's `check` run, with the harness's rules and procedure
   ([harness.md](./harness.md)). From this landing on, a worker writes notes; nothing yet fails it for not writing one.
2. **The gate's trigger.** `blizzard-change-notes owed` as `gate / change notes` on pull requests, and the matrix row.
   It lands after the format so the check never fails a change for a note its author had no way to write.
3. **Deprecations and the major's guide.** `removes-in`, `retires`, `postpones`, and `guide`, proven against a synthetic
   history of tags in the tool's tests, since no major exists to prove it on. Before 1.0 it gathers for every minor a
   deprecation names ([upgrade.md](./upgrade.md) §A major's guide).

The epic lands after `epic:release-engineering/first-release`, which settles how `docs/upgrade.md`'s boundaries carry
their versions and leaves a release for the first cut to count from. It does not wait on
`epic:release-engineering/compat-floor`: the breaking pair reads the `!` marker compat-floor already reads, so each
check holds its own half whichever lands first, and the `also in` marker reads plain tags until compat-floor's
supported-release resolution replaces them. `flag` is refused until `epic:release-engineering/dark-launch` lands its
registry.

## The first assembled release

The changes landed between the last release and the first landing have no notes. That landing writes them: the agent
landing it reads the commit log since the last release tag and writes an unreleased note for each change an operator
would notice, under `bzh:change-note`, and an upgrade section for every operator-facing step the
`## Landing and
deployment` sections of the issues that range closed carry, so the first assembled release is complete.
The commit appendix covers anything missed.

## Landing and deployment steps

Until this epic lands, a release's deployment steps live in its issues: `epic:release-engineering/first-release`'s
runbook reads the `## Landing and deployment` sections of the issues a release's range closed beside its **Upgrade
notes** ([runner-pin.md](../../first-release/spec/runner-pin.md)). From the first landing, an operator-facing step
travels as a note's upgrade section instead ([upgrade.md](./upgrade.md) §The upgrade section), and once the first
assembled release is published the runbook reads its **Upgrade notes** alone. An issue's `## Landing and deployment`
section keeps only what is not the operator's — a step for the landing itself — and says nothing a release's operator
must do.

## What moves over

- **`scripts/release-notes.sh`** stays, as the commit appendix's source. Its placeholder **Upgrade notes** heading and
  the breaking-changes section it builds from `!` subjects are removed — the notes own both now — and
  `tests/test_release_notes.py` follows.
- **`docs/versioning.md`**'s paragraph saying release notes are generated from Conventional Commits points to
  `changes/README.md` instead; `docs/ci.md`'s release section names the render step; `docs/upgrade.md`'s §Before you
  pull reads the release's upgrade notes as it does today, now that they are written.
- **`docs/upgrade.md`'s existing boundaries** stay exactly where they are. No note is written for a boundary a release
  already carried.
- **Releases already published** keep the notes they were published with. No tag is re-rendered.
- **`bzh:release`'s `gh release edit`** after the tag goes, with the placeholder it filled. Releases are immutable once
  published (`epic:release-engineering/supply-chain`), and only a corrected body is ever re-rendered onto one
  ([assembly.md](./assembly.md) §The release workflow).
