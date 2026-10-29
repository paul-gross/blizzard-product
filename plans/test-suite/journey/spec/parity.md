# Wire parity in CI

Paths here are relative to the `blizzard-mock` repo root.

## The repair

`tests/test_wire_parity.py` runs `mock-opencode`'s output through blizzard's `parse_run_jsonl`, loading the parser from
blizzard's source file and stubbing the one module it used to import. Since blizzard's
`src/blizzard/runner/harness/opencode/shapes.py` began importing `blizzard.runner.harness.json_value`, ten of the
module's tests fail with `ModuleNotFoundError`. The parser runs instead under blizzard's own environment, as a
subprocess of blizzard's virtual environment fed the façade's output, so its imports resolve to whatever they become and
the guard stubs nothing. The subprocess prints each parsed event as one JSON object per line, its type and the
attributes the tests assert on, or, when the parser refuses the stream, one object naming `OpenCodeShapeError` and its
message, and the tests assert on that. The repair lands first and on its own.

The two tests in the module that read nothing of blizzard's,
`test_no_mirror_entry_point_takes_transposable_positional_arguments` and
`test_permission_denial_refuses_on_a_non_opencode_wire`, lose the module's `needs_blizzard` marker, so the gate runs
them already.

## The ref

`.github/actions/resolve-blizzard-ref/` mirrors blizzard's `resolve-mock-ref`, by the same rules from the other side: a
`v*` tag checks out blizzard's tag of that name; a pull request checks out blizzard's branch of the head's name if it
exists, else blizzard's line branch, `release/X.Y` or `master`; a push to `master` checks out `master`. Its script is
unit-tested as blizzard's is. A wire change made in both repositories on branches of one name is then proved together,
whichever lands first.

## The jobs

`.github/workflows/gate.yml` gains a `parity` job: checkout of `blizzard-mock` and of blizzard at the resolved ref
beside it, `uv sync` in both, and `mise run test -- -m needs_blizzard` with `BLIZZARD_SOURCE` and `BLIZZARD_RUNNER_BIN`
pointing at the checkout. It runs both `needs_blizzard` modules, the parity guard and the real runner against the mock
hub, the second taking about a minute. Both repositories are public, so the checkout needs no token. The job lands
optional and is made required by its own later change, once it has run green, as `gate / wire parity (needs_blizzard)`.

The resolver, like blizzard's, refuses any event its rules do not name, so it gains one input: an explicit ref, which
skips resolution. A scheduled workflow, `nightly.yml`, calls the parity job nightly with that input set to `master`, and
so catches a wire change merged in blizzard with no mock branch of its name, which no pull request of the mock's would
see. `gate.yml` is called by workflows, never triggered itself, so the schedule lives in its own caller.

A wire change lands in blizzard first, then in the mock. Until the mock's branch of the same name merges, the mock's own
pull requests and its `master` see the new wire through the resolver or the nightly run and fail the parity check. That
gap is expected, it is the guard doing its work, and the mock's change closes it.

## Citations

blizzard-context's `verification/blizzard/commands/mock.md`, which lists the mock's required checks and calls the guard
local-only, and `companion-changes.md`, which states `bzh:wire-change-extends-mock`, name the job and the matching
branch as the way a wire change and its mirror land together. `gate.yml`'s header comment and the README's account of
the guard as a local-only gate change with them.
