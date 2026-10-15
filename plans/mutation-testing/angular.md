---
epic: mutation-testing/angular
refinement: refined
status: horizon
---

# Plan — `epic:mutation-testing/angular`

**Third epic — the Angular workspace.** Blocked on tooling outside this repo's control; the section below records the
research and the stance.

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
