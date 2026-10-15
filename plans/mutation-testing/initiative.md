---
initiative: mutation-testing
refinement: refined
epics:
  - diff-scoped
  - corpus-sweep
  - angular
---

# Plan — `initiative:mutation-testing`

A fleet that ships while its owner sleeps acts on one signal: the suite went green. A human reviewer carries a private
doubt about that signal — *would these tests actually have noticed?* — and blizzard's reviewers already act on the doubt
by hand: mutate a suspicious line, re-run the tier, name the assertion that fired (`bzh:mutation-review-selection`).
That practice is judgment applied one line at a time, and it reaches only the lines a reviewer thought to suspect. This
initiative gives the doubt a machine. Mutation testing seeds small defects — a flipped comparison, a dropped guard, a
negated condition — one at a time, and re-runs the suite against each; a seeded defect the suite survives is a place the
product can regress with the gate still green. Coverage says a line ran; a killed mutant says its behavior is pinned.

## The shape of the constraint

Only the fast tiers can sit under a mutation tool. The Python suite's unit and component tiers (`uv run pytest`,
xdist-parallelized) re-run cheaply; the service, e2e, crash, and journey tiers spawn real daemons and fixture
workspaces, and re-running those per mutant — thousands of mutants, minutes apiece — is not a cost any cadence absorbs.
The backend corpus is roughly 48k lines under `src/blizzard/`; the frontend is four Angular projects (~120 spec files)
running vitest through the Angular CLI's `@angular/build:unit-test` builder. Whatever runs must land as a
verifiability-matrix row with a stable method id (`bzh:verifiability-matrix`), like every other way blizzard asserts
correctness.

## Alternatives considered

Four shapes were weighed before the initiative was cut into its epics:

1. **Diff-scoped, advisory, in the delivery path** — chosen as the first epic, [`diff-scoped`](./diff-scoped.md): the
   feedback reaches the change that introduced the weak test, while it is still open.
2. **A full-corpus sweep only** — kept, but second, as [`corpus-sweep`](./corpus-sweep.md), and folded into gardening
   rather than built as its own schedule: its findings are disconnected from any open change, and a first run surfaces
   hundreds of survivors that need a standing triage policy before they are signal.
3. **Agent-driven selection only, no tool** — formalizing `bzh:mutation-review-selection` into a review-time step and
   stopping there. Highest signal per mutant and zero new dependencies, but it only reaches lines someone suspected; it
   stays as the standing complement to the tooling, and as the only path currently open on the frontend
   ([`angular`](./angular.md)).
4. **A full-corpus blocking gate** — rejected: the runtime would sit in every delivery's path for a signal that mostly
   concerns unchanged code.

## Open questions

- The equivalent-mutant policy: suppression list, score threshold, or per-file ignores — decided before any run is
  allowed to block.
