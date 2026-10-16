# When a note is owed

A note is owed by every change an operator would notice. Two layers hold that: the harness's rule, which a worker
applies while writing and the review node applies while reviewing, and a mechanical check in the gate beneath it for
what a diff can prove.

## The judgement

`bzh:change-note` ([harness.md](./harness.md)) defines *noticeable* as anything the people who run blizzard can see,
set, or must do differently after taking the release: a CLI verb, flag, or output; a configuration key or its default;
an HTTP route or field; a board surface they use; how the fleet claims, lands, spends, or recovers; a fix to something
they could have hit; a change to an operator document's procedure. It is not owed for a refactor, a test, a CI or
tooling change, a change to `blizzard-context`, or a fix to a defect that never reached a release. One note is written
per noticeable change, not per commit or per chunk.

The worker making the change writes the note in the same change, following `bzh:change-notes`
([harness.md](./harness.md)). The review node holds it to `bzh:change-note` as it holds the code to every other
standard: a missing note, a note that describes the implementation, or a `fixed` note that names the cause rather than
the symptom is a finding like any other.

## The mechanical check

`blizzard-change-notes owed --against <ref>` diffs the pull request's range against its merge-base, the way
`blizzard-compat` does on a pull request, and fails when the range owes a note it does not add. It runs in `gate.yml` on
pull requests only, as `gate / change notes`, and `mise run change-notes` reproduces it locally against `origin/master`.
A push to `master` or a release branch runs only `check` ([fragments.md](./fragments.md) §Validation): by then the range
has already been judged.

### The surface trigger

A range that changes any contract document `epic:release-engineering/compat-floor` diffs owes at least one added or
edited unreleased note. The documents are the ones its surfaces table lists
([surfaces.md](../../compat-floor/spec/surfaces.md)) — both OpenAPI documents for the wire and the HTTP API, the
configuration, trace, and egress contracts — which the tool reads from compat-floor's surface registry in
`src/blizzard/tools/compat/` rather than listing them itself, so a surface that joins the version scheme joins the
trigger in the same change. Two adjustments are the trigger's own: it adds `contracts/cli/hub.json` and
`contracts/cli/runner.json`, since a changed verb or flag is the first thing an operator notices though no version rule
governs it, and it leaves out the store schema's documents, since a revision that walks back is invisible to the
operator and one that does not is a break, caught by the breaking pair below.

A range whose contract change an operator would not notice — a reordered enum, a corrected description — waives the
trigger with a trailer on any commit in the range, carrying a reason:

```text
Change-Note: none — reorders the CLI contract's commands, no surface change
```

The tool reports each waiver it honours, so the review node reads the reason. A waiver without a reason fails.

### The breaking pair

The `!` marker and `breaking: true` travel together, in both directions:

- a range carrying a commit whose subject has the `!` marker (`docs/versioning.md`; the same pattern as the marker
  `blizzard-compat` reads, in `src/blizzard/tools/compat/`) owes an added unreleased note with `breaking: true` and an
  `## Upgrade` section;
- a range adding a note with `breaking: true` owes a `!`-marked commit.

No trailer waives either. Whether the break may pass at all is `epic:release-engineering/compat-floor`'s check, which
reads the marker; this check guarantees only that every acknowledged break reaches the operator with its upgrade note.

### Released notes stay put

The same pass refuses a range that renames, deletes, or changes the frontmatter of a note under `changes/<X.Y.Z>/`, or
adds one there, unless the range is a cut — `epic:release-engineering/release-lines`' cut commit, identified by its
`Release-Cut:
<version>` trailer ([train.md](../../release-lines/spec/train.md) §The cut commit), in which
`blizzard-change-notes cut` moved them ([assembly.md](./assembly.md) §The cut).
