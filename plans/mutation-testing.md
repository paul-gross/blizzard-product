# Plan — `epic:mutation-testing`

A fleet that ships while its owner sleeps acts on one signal: the suite went green. A human reviewer carries a private
doubt about that signal — *would these tests actually have noticed?* — and blizzard's reviewers already act on the doubt
by hand: mutate a suspicious line, re-run the tier, name the assertion that fired (`bzh:mutation-review-selection`).
That practice is judgment applied one line at a time, and it reaches only the lines a reviewer thought to suspect. This
epic gives the doubt a machine. Mutation testing seeds small defects — a flipped comparison, a dropped guard, a negated
condition — one at a time, and re-runs the suite against each; a seeded defect the suite survives is a place the product
can regress with the gate still green. Coverage says a line ran; a killed mutant says its behavior is pinned.

## The shape of the constraint

Only the fast tiers can sit under a mutation tool. The Python suite's unit and component tiers (`uv run pytest`,
xdist-parallelized) re-run cheaply; the service, e2e, crash, and journey tiers spawn real daemons and fixture
workspaces, and re-running those per mutant — thousands of mutants, minutes apiece — is not a cost any cadence absorbs.
The backend corpus is roughly 48k lines under `src/blizzard/`; the frontend is four Angular projects (~120 spec files)
running vitest through the Angular CLI's `@angular/build:unit-test` builder. Whatever runs must land as a
verifiability-matrix row with a stable method id (`bzh:verifiability-matrix`), like every other way blizzard asserts
correctness.

## What to build

**First slice — the changed lines, at delivery time (backend).** [mutmut](https://mutmut.readthedocs.io/) as a dev
dependency, scoped to the lines a branch changed, running the unit + component tiers against each mutant. mutmut
selects, per mutant, only the tests whose coverage touches the mutated line, which is what makes a per-delivery run
affordable. It lands as a `mise run mutation` task and a `blizzard:mutation` matrix row, excluding the generated
migration trees. The signal is advisory first: surviving mutants are reported on the delivery, and nothing blocks until
the noise level of real runs is known. The harness rule completes the loop — a survivor on a high-value line is answered
with a named assertion that kills it, per `bzh:mutation-review-selection`, not with a shrug at an aggregate score.

**Second slice — the whole corpus, on a cadence.** A scheduled full sweep over `src/blizzard/` — resumable,
session-based tooling ([cosmic-ray](https://cosmic-ray.readthedocs.io/), or mutmut's cache if it proves sufficient) —
whose survivors become filed issues the fleet ingests as ordinary work. The fleet strengthening its own test suite
overnight is the product eating its own cooking; what makes it honest is a triage policy for equivalent mutants
(behavior genuinely unchanged), so the report converges instead of going stale after its first run.

**Third slice — the Angular workspace.** Blocked on tooling outside this repo's control; the section below records the
research and the stance.

## Alternatives considered

Four shapes were weighed before the slices above were cut:

1. **Diff-scoped, advisory, in the delivery path** — chosen as the first slice: the feedback reaches the change that
   introduced the weak test, while it is still open.
2. **A scheduled full-corpus sweep only** — kept, but second: its findings are disconnected from any open change, and a
   first run over 48k lines surfaces hundreds of survivors that need a standing triage policy before they are signal.
3. **Agent-driven selection only, no tool** — formalizing `bzh:mutation-review-selection` into a review-time step and
   stopping there. Highest signal per mutant and zero new dependencies, but it only reaches lines someone suspected; it
   stays as the standing complement to the tooling, and as the only path currently open on the frontend.
4. **A full-corpus blocking gate** — rejected: the runtime would sit in every delivery's path for a signal that mostly
   concerns unchanged code.

## The Angular surface

As of August 2026 the tooling path is closed at the seam blizzard sits on. The frontend's four projects run vitest
through Angular's own `@angular/build:unit-test` builder, and [StrykerJS](https://stryker-mutator.io/) — the one serious
mutation tool for TypeScript — cannot drive it: its
[vitest runner](https://stryker-mutator.io/docs/stryker-js/vitest-runner/) needs to load and extend a vitest
configuration programmatically, an API the Angular builder does not expose. The gap is an open StrykerJS feature request
([stryker-js#5655](https://github.com/stryker-mutator/stryker-js/issues/5655), November 2025, no maintainer activity
yet), and Stryker's [official Angular guide](https://stryker-mutator.io/docs/stryker-js/guides/angular/) still assumes
Karma.

A bypass exists: a standalone per-project vitest config compiled through
[`@analogjs/vite-plugin-angular`](https://analogjs.org/docs/features/testing/vitest), the setup Stryker's vitest runner
is known to work with. It is not worth its price here — a second test configuration drifting beside `angular.json`,
maintained for the mutation tool alone. Two of the vitest runner's standing limits would also apply when the seam opens:
no browser mode (immaterial — the real-Chromium `shell-sweep` specs are already excluded from `ng test`) and
single-threaded runs per Stryker worker.

The stance: do not build the bypass. Frontend mutation stays agent-driven through `ng test` under
`bzh:mutation-review-selection` for now; adopt Stryker with a `web:mutation` matrix row when stryker-js#5655 closes or
Angular's builder grows the programmatic seam.

## Open questions

- The per-delivery runtime and survivor noise of the first diff-scoped runs — a timed spike over one module sizes both
  before anything is wired into the delivery path.
- The equivalent-mutant policy: suppression list, score threshold, or per-file ignores — decided before any run is
  allowed to block.
- Whether the scheduled sweep files issues autonomously or a human triages its survivors into the intake queue.
