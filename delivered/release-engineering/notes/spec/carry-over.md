# Carry-over

The epic lands into a repository whose releases have been described by their commit logs, whose upgrade guide is a
document of boundaries, and whose release procedure, `workspace:/context/project/workflows/release.md`, ends with a
hand-written edit. None of it is thrown away.

## The order of landings

The epic lands in two phases, split by one line: the first release `epic:release-engineering/first-release` publishes.
That release, `v0.2.0`, is cut with `scripts/release-notes.sh` and its **Upgrade notes** placeholder replaced by hand
with a pointer to `docs/upgrade.md` §Upgrading to 0.2.0, so nothing may take the placeholder away, or tell a worker to
write a note, until it is published.

**The tooling, ahead of the first release.** Each of these changes nothing anyone runs: no worker is told to write a
note and nothing publishes from one, so each lands as soon as what it reads exists.

1. **The format and its validator.** `changes/` with its `README.md` and an empty `changes/unreleased/`,
   `blizzard-change-notes check`, and `gate / change notes` running `check` on every trigger
   ([fragments.md](./fragments.md)). `flag` is refused until step 4.
2. **The assembler.** `cut` and `render`, built against fixture trees and synthetic tags, with `also in` read from plain
   release tags ([assembly.md](./assembly.md)). It follows first-release's rewrite of `docs/upgrade.md` into per-release
   `## Upgrading to` sections, which `cut` files boundaries under ([upgrade.md](./upgrade.md) §Boundaries).
   `release.yml` does not call it yet.
3. **Deprecations and the major's guide.** The cut's refusals for `removes-in` and `postpones`, and `guide`, proven
   against a synthetic history of tags, since no major exists to prove them on ([upgrade.md](./upgrade.md)).
4. **The flag field.** `flag` admitted, held notes, and **Turned on**, once `epic:release-engineering/dark-launch`'s
   registry exists ([assembly.md](./assembly.md) §Held notes).

**The switch, once `v0.2.0` is published.**

5. **Publishing from notes, and requiring them.** `release.yml` publishes `render`'s body, `scripts/release-notes.sh`
   shrinks to the commit appendix, `docs/upgrade.md` gains its `## Upgrading to the next release` heading, and the
   harness's rules, procedure, and `blizzard:change-notes` row land ([harness.md](./harness.md)), with the notes since
   `v0.2.0` written beside them (§The first assembled release). From this landing on, every worker writes notes. With
   it, by hand, the operator changes `workspace:/context/project/workflows/release.md` to run `cut` in
   `epic:release-engineering/release-lines`' cut commit, when the instance moves onto a build carrying the landing and
   in any case before the next release is cut ([harness.md](./harness.md) §Cutting a release).
6. **The gate's trigger.** `blizzard-change-notes owed` on pull requests ([owed.md](./owed.md)). It reads the contract
   documents from `epic:release-engineering/compat-floor`'s surface registry and the `!` marker as `blizzard-compat`
   reads it, so it lands once `blizzard-compat` has, which itself waits for a release to compare against; and after step
   5, so it never fails a change for a note its author had not yet been told to write.
7. **The cross-check.** The cut's and the release workflow's match of `compat-breaks.json` against the breaking notes,
   and `also in` over the supported releases ([assembly.md](./assembly.md) §Breaks cross-checked), once
   `blizzard-compat` writes the file at a tag.

## The first assembled release

The changes landed between `v0.2.0` and step 5 have no notes. Step 5 writes them: the agent landing it reads the commit
log `v0.2.0..HEAD` and writes an unreleased note for each change an operator would notice, under `bzh:change-note`, and
an upgrade section for every operator-facing step the `## Landing and deployment` sections of the issues that range
closed carry, so the first assembled release is complete. The commit appendix covers anything missed.

## Landing and deployment steps

Until step 5, a release's deployment steps live in its issues: `epic:release-engineering/first-release`'s runbook reads
the `## Landing and deployment` sections of the issues a release's range closed beside its **Upgrade notes**
([runner-pin.md](../../first-release/spec/runner-pin.md)). From step 5, an operator-facing step travels as a note's
upgrade section instead ([upgrade.md](./upgrade.md) §The upgrade section). Once the first assembled release is
published, the workspace's runbook — `context/project/post-delivery.md` and the release binding in
`context/project/local-instance.md` — is rewritten to read a release's **Upgrade notes** alone. That is an attended
commit to the workspace repository, which no fleet chunk can land. An issue's `## Landing and deployment` section keeps
only what is not the operator's — a step for the landing itself — and says nothing a release's operator must do.

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
- **`workspace:/context/project/workflows/release.md`'s `gh release edit`** after the tag goes, with the placeholder it
  filled. Releases are immutable once published (`epic:release-engineering/supply-chain`), and only a corrected body is
  ever re-rendered onto one ([assembly.md](./assembly.md) §The release workflow).
