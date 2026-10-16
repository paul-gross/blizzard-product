# Assembling a release's notes

## The cut

`blizzard-change-notes cut <X.Y.Z>` runs in `epic:release-engineering/release-lines`' cut commit, the
release-preparation commit carrying the trailer `Release-Cut: <X.Y.Z>` at which every candidate and patch is tagged
([train.md](../../release-lines/spec/train.md) §The cut commit). Which branch that commit lands on, and whether it moves
`pyproject.toml`'s version — a further candidate's, on `release/X.Y`, does not — are release-lines'. In that commit, it:

1. validates `changes/` as `check` does ([fragments.md](./fragments.md) §Validation), and refuses a deprecation whose
   `removes-in` is not later than `X.Y` ([upgrade.md](./upgrade.md) §Deprecations);
2. moves every note in `changes/unreleased/` into `changes/<X.Y.Z>/`, except held notes (§Held notes);
3. moves every `docs/upgrade.md` boundary a moved note's `upgrade-guide` names from `## Upgrading to the next release`
   into `## Upgrading to <X.Y.Z>` ([upgrade.md](./upgrade.md) §Boundaries);
4. for a major, runs the guide and refuses on an unretired deprecation ([upgrade.md](./upgrade.md) §A major's guide);
5. runs `epic:release-engineering/compat-floor`'s tag-mode check over the cut commit and refuses when a break it would
   let through is named by no `breaks` entry of a breaking note in `changes/<X.Y.Z>/` (§Breaks cross-checked).

A second candidate for the same version (`-rc.2`) runs the cut again and adds what has landed on its line since into the
same directory. A cut with nothing to move still succeeds; its release is notes-empty and says so.

## Rendering

`blizzard-change-notes render <tag>` reads the tag's tree and writes the published notes as Markdown. It is pure over
that tree, the previous release tag's tree, the commit range, and the tag's `compat-breaks.json`, so rendering a tag
twice gives the same notes. The order:

| Section                                         | Holds                                                                                                                                                                                                                                                         |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Upgrading to X** (a major)                    | The gathered guide ([upgrade.md](./upgrade.md) §A major's guide).                                                                                                                                                                                             |
| **Upgrade notes**                               | Every upgrade section in the version's directory, each headed by its note's first sentence and linked to its boundary, then the breaks `compat-breaks.json` records, each beside the note that names it (§Breaks cross-checked). Omitted when there are none. |
| **Turned on**                                   | Every flag whose `default` is `on` in this tag's registry and `off` in the previous release tag's, by its registry `summary` (§Held notes).                                                                                                                   |
| **Breaking changes**                            | Every `breaking: true` note's body.                                                                                                                                                                                                                           |
| **Deprecated**                                  | Every `deprecated` note, each ending with the line that removes it.                                                                                                                                                                                           |
| **Added**, **Changed**, **Fixed**, **Security** | The remaining notes by kind; `removed` notes that do not break appear under **Changed**.                                                                                                                                                                      |
| **Commits**                                     | `scripts/release-notes.sh --range <previous tag>..<tag>`'s output inside a collapsed `<details>` block.                                                                                                                                                       |

Within a section notes are ordered by id, so the output never depends on the order files were moved. A note whose id
also appears under an earlier release's directory on another line — a fix carried back as a patch — is marked with that
release (`also in 0.7.1`), read from the supported releases' tags that `epic:release-engineering/compat-floor` resolves.

## Breaks cross-checked

At a tag, `epic:release-engineering/compat-floor` writes every break it let through to `compat-breaks.json`, one entry
per finding with its surface, `where`, and the release it breaks, and the release workflow attaches it
([gates.md](../../compat-floor/spec/gates.md)). The notes and that file must agree: every entry is named by a note in
the version's directory with `breaking: true`, the entry's surface among its `surfaces`, and its `where` among its
`breaks` ([fragments.md](./fragments.md) §Fields). The cut runs the same check over the cut commit so a mismatch is
found before the tag; render repeats it over the attached file as the backstop, and an entry no note names fails the
release before anything is published. A breaking note that names a `where` no entry records is reported, not failed: a
break the static diff cannot see, such as a behavior an upgrade note warns of, is still the operator's to read.

## The release workflow

`.github/workflows/release.yml`'s `generate release notes` step runs `blizzard-change-notes render "${GITHUB_REF_NAME}"`
in place of `scripts/release-notes.sh` alone, keeping the same `PREV_TAG` resolution for the commit range, after the
step that writes `compat-breaks.json`. A tag whose tree has no `changes/<version>/` — any release cut before this epic —
renders the commit appendix alone.

The rendered notes become the release's body at publish time, in the `gh release create --notes-file` that publishes it;
they are never an attached file. Releases are immutable once published (`epic:release-engineering/supply-chain`'s
[artifacts.md](../../supply-chain/spec/artifacts.md)): nothing edits, adds, or replaces an attached file afterwards, and
`bzh:release` no longer asks anyone to finish a published release by hand ([harness.md](./harness.md)). The body is the
one thing that may change after publishing: a released note's body corrected afterwards ([fragments.md](./fragments.md)
§Once released) reaches the published release by re-rendering that tag and `gh release edit --notes-file`, which touches
the body alone.

## Lines

A fix carried back to a released line by cherry-pick brings its unreleased note with it, under the same id, so the
patch's cut moves it into `changes/<X.Y.Z+1>/` on the branch while `master`'s copy waits for the next minor. Each line's
notes are complete for a reader of that line; the `also in` marker tells a reader of the next minor they have already
seen it. A fix written on an old line alone, for code `master` has removed, writes its note on that line alone.

## Held notes

A note carrying `flag` stays in `changes/unreleased/` at a cut while the flag ships off by default at the cut commit,
and is moved by the first cut at which it ships on. The assembler reads a flag's `default` and `summary` from
`epic:release-engineering/dark-launch`'s registry, `src/blizzard/foundation/flags/registry.toml`
([registry.md](../../dark-launch/spec/registry.md)), at the cut commit and, for **Turned on**, at the previous release
tag with `git show <tag>:src/blizzard/foundation/flags/registry.toml`, never importing an old release's code. `check`
refuses a `flag` that names no flag in the registry; until the registry exists, it refuses the field.
