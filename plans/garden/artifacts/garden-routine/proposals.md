# Proposal entries

A `docket` asset is a list. A proposal is a proposed response to one or more findings — never an observation, and never
work in itself until a person makes it so.

```json
{
  "ref": "P1",
  "kind": "prevent",
  "title": "Author a comment standard covering change-history narration",
  "body": "…the case, in enough detail that a person can decide without re-reading the findings…",
  "findings": ["fin_01JKQ8Z3M4N5P6R7S8T9V0W1X2", "fin_01JKQ8Z3M4N5P6R7S8T9V0W1X3"]
}
```

- **kind** — one of `remediate`, `prevent`, `mechanize`. The hub stores it and never interprets it; it exists so an axis
  can be asked whether it is being weeded or actually repaired.
- **findings** — the ids this response answers. Required and non-empty: a proposal with no findings behind it is an
  opinion the run was not asked for.

## The three kinds

- **remediate** — do the cleanup the findings describe. The campaign: one proposal per theme per area, sized as work
  somebody could actually pick up, with its findings linked rather than pasted into the body.
- **prevent** — fix the source so the inflow stops. The missing standard, the exemplar spreading the pattern, the rule
  too fuzzy to follow. Usually the highest-leverage of the three, and the first thing to reach for when a sweep returns
  a great many findings: filing a thousand cleanups while the thing producing them runs untouched is motion, not
  progress.
- **mechanize** — retire the judgment itself, moving the check into a lint rule, a config, or a gate so no later run
  pays a model to notice this class again. Carries its case: which finding class it retires, and at what run-cost saved.

## What happens to one

Nothing, until a person acts. A proposal attaches to its axis and waits. **Passing** on it records that it was
considered and declined, so a later run's reconcile does not raise it again as though it were new. **Accepting** it
creates a work item linked to the proposal in the same act — and that link is what lets the axis ever answer whether the
work that landed resolved the findings that prompted it.
