---
epic: mutation-testing/diff-scoped
refinement: pristine
status: delivered
---

# Plan — `epic:mutation-testing/diff-scoped`

**First slice — the changed lines, at delivery time (backend).** [mutmut](https://mutmut.readthedocs.io/) as a dev
dependency, scoped to the lines a branch changed, running the unit + component tiers against each mutant. mutmut
selects, per mutant, only the tests whose coverage touches the mutated line, which is what makes a per-delivery run
affordable. It lands as a `mise run mutation` task and a `blizzard:mutation` matrix row, excluding the generated
migration trees. The signal is advisory first: surviving mutants are reported on the delivery, and nothing blocks until
the noise level of real runs is known. The harness rule completes the loop — a survivor on a high-value line is answered
with a named assertion that kills it, per `bzh:mutation-review-selection`, not with a shrug at an aggregate score.

## Open questions

- The per-delivery runtime and survivor noise of the first diff-scoped runs — a timed spike over one module sizes both
  before anything is wired into the delivery path.
