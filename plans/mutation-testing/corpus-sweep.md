---
epic: mutation-testing/corpus-sweep
refinement: pristine
status: delivered
---

# Plan — `epic:mutation-testing/corpus-sweep`

**Second slice — the whole corpus, as a gardening axis.** The sweep is not a new routine: it is an axis in
blizzard-context's garden registry, run by the gardening pass blizzard already has, standing on the first slice's
`mise run mutation` task. The pass brings the method — resolve the axis, sweep a scope, record the measurement, draft
the work its findings earn for the operator's approval — so all this slice adds is the axis entry itself:

- **Evaluates** — suspected weak tests: behavior the suite executes but does not pin, found as mutants that survive it.
- **Scope** — the same slugs the `architecture` axis narrows to on the backend (`hub-daemon`, `runner-daemon`,
  `shared-spine`, `cli-surface`), each resolving to the source paths a run mutates. A slug is what keeps a sweep
  affordable: one run mutates one daemon, and the next run another, where a single pass over 48k lines would not fit a
  night.
- **Criteria** — `bzh:mutation-review-selection`, whose detection clause gains the command a run executes. The standard
  for telling one survivor from another is absent; authoring it is the axis's first act.
- **Measurement** — mutants generated, killed, and surviving per scope, with the survivors that no triage has classified
  — the number that has to fall for the sweep to be converging.

A survivor is a *suspect*, not yet a finding, and reading it is where the model earns its place. It resolves to one of
four things: a test that reaches the line but asserts nothing about it (name the test, and the assertion that would kill
the mutant), a line no gating test reaches at all, code the mutation shows is superfluous (a guard that cannot fire is a
deletion, not a new test), or an equivalent mutant whose behavior is genuinely unchanged — suppressed once, so the next
run does not re-litigate it. The first three become the pass's proposals: strengthened assertions, new cases, removed
code, each filed as ordinary work the fleet picks up. The pass itself edits nothing.

This is its own axis, `mutation-testing`, beside the `tests` axis the garden plan names: `tests` prunes what the suite
carries in excess, this one finds what it fails to pin. The kill map a sweep leaves behind — which test kills which
mutants — is also the measured evidence `tests` was promised, since a test that kills nothing another test does not is
dead weight. Until `epic:cadence` lands, an operator starts a sweep the way they start any gardening pass; once it does,
the axis's routine declares its interval there instead of this epic building a timer.
