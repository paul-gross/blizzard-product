# Finding entries

A run's `survey` and `delta` assets are **lists**, serialized as JSON. The hub validates the shape below and reads
nothing inside it — what a finding means is the routine's business, not the platform's.

## A candidate finding (`survey`)

The survey has no ids to work with, because identity is minted at delivery. Each candidate carries a local `ref`, stable
only within this submission, so the reconcile step can name it.

```json
{
  "ref": "F1",
  "class": "retrospective-comment",
  "locus": "blizzard/src/blizzard/runner/lease.py:42",
  "summary": "Module docstring narrates the change history: \"Previously the lease expired at…\".",
  "introduced": "a1b2c3d"
}
```

- **class** — the routine's own vocabulary for a kind of weed. The hub indexes it and never interprets it: it can count
  how often a class recurs, which is what a mechanization case rests on, without knowing what the name means. Keep the
  spelling stable across runs — two spellings of one class split its own recurrence count and hide the case.
- **locus** — where it lives. A repo-relative path, optionally `:line` or `::symbol`.
- **introduced** — best effort. The commit that introduced what the finding objects to, from `blame` on the locus. Omit
  it rather than guess: a reformat defeats blame, and a rule that went stale because the code around it moved has no
  introducing commit at all.

**A finding is one instance — one thing somebody could fix.** Not a theme, and not a tally: seventeen retrospective
comments in the lease package are seventeen findings, each with its own locus and its own id. That is what lets six of
them resolve while eleven stay open, what lets a campaign hand each one to its own cheap agent, and what makes an axis's
outflow a real count rather than a theme that stays open until the last instance falls.

Grouping is not this artifact's job. A run that finds seventeen of anything says so seventeen times here and once in the
docket, because the entity built for "these belong together" is the proposal, which names every finding behind it.

## A delta (`delta`)

The delta is what the axis's state changes by — never a restatement of the state itself. Two entry kinds.

**Additions** carry a candidate through unchanged, minus its `ref`. The hub mints a `fin_<ULID>` for each.

```json
{ "op": "add", "class": "retrospective-comment", "locus": "…", "summary": "…" }
```

**Transformations** act on a finding already live on this axis, named by its id.

```json
{ "op": "observed", "id": "fin_01JKQ8Z3M4N5P6R7S8T9V0W1X2" }
{ "op": "gone",     "id": "fin_01JKQ8Z3M4N5P6R7S8T9V0W1X3", "note": "No longer reproduces at the recorded locus." }
```

- **observed** — still reproduces. Restamps when it was last seen and against which revision. It carries no payload: the
  finding was true when it was recorded and it is true now, so there is nothing to revise.
- **gone** — looked for, not found. This does not close the finding; it flags it for a person, because a finding leaves
  the live set on human judgment, never on a pass's word.

Each delivered list declares the **scope** it was swept under — one of the names the routine's strategy defines. The hub
stamps that scope on every finding the list mints and never resolves what it names, so an axis's findings partition into
buckets a later run can fetch one of. A finding keeps the scope it was minted under, and a fanned-out run delivering one
list per partition delivers one scope per list.

Every run also records the revision it read, per repo, and the measurement its axis declares. All of these are declared
on the artifact rather than on any finding inside it, and recorded whether or not the delta contains a single entry — a
clean run's product is its datapoint.

## The rule the delta exists to enforce

**Emit transformations only for ground this run actually visited.** A finding the run did not look at gets no entry, and
so keeps its last word. Silence about a finding is never a claim about it, which is what lets a scoped or delta-mode run
be honest without anyone having to remember what it skipped.
