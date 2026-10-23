# Slow outliers

## Measure first

The plan's figures for `tests/test_runner_harness_opencode_diagnostic.py`, about 164 seconds of CPU, one 35-second test,
and some thirty at four to five seconds, do not match the file as it stands: it holds twenty test functions, twenty-four
items with one parametrized five ways, and its subprocess tests carry bounds of five seconds and fixed sleeps of at most
0.7. The cost is either in what the tests start, the pty fork, the Landlock probe, the six megabytes each pipe drains,
the `CompatibilityDiagnostic` runs, or in a recent change that already cut it.

So the work opens with a measurement on the CI shape: the unit and component tiers run once on an `ubuntu-latest` runner
with `--durations=0`, through a `workflow_dispatch` input on `gate.yml` that adds the flag, and the slowest fifty items
are recorded in this epic's plan, beside the baseline. The durations recorder
([pipeline.md](./pipeline.md#recording-durations)) gives the same numbers on every run thereafter. Whatever the
measurement names as the outliers, this file among them or not, is what the rest of this leaf applies to.

## Deciding per test

For each outlier, the question is whether the slow thing is the thing under test:

- **When it is**, the cost is the test's, and the test stays. A test that proves a process group is reaped must start a
  process group. Its cost is reduced only by doing less of the same thing, such as draining one megabyte through each
  pipe where six prove nothing more, and only where the smaller input still exceeds every buffer the assertion is about.
- **When it is not**, the slow part is replaced by the fake already in the file or by an in-memory double, and the test
  moves to the unit tier if nothing slow remains. A test whose subject is how the adapter parses a session's output,
  say, has no business spawning the session.

The deciding is recorded per test in the change that makes it, in the commit message, not in the file.

## Sharing the fake binary

`_stub_binary` writes a two-line `fake-opencode` script into each test's `tmp_path` through `_probe()`, and the script
is never executed. A module-scoped fixture writes it once into a `tmp_path_factory` directory, and `_probe()` takes it
from there. Nothing a test does writes to the binary, so sharing it changes no test's subject. Any test found to modify
or execute it keeps its own.

The same applies to any other outlier the measurement names that builds an identical fixture per test: it is shared at
the widest scope at which no test mutates it.
