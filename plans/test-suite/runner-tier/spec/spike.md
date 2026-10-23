# The runner side of the spike

The foundation's spike proved the hub side and never reached a runner. Before the stepping host is built, a throwaway
spike runs two tests from Playwright against a real `blizzard-runner host` and a mock hub, on today's surfaces, to size
what a host costs and to find where the surfaces below fall short.

## What it ports

- **One runner-side test.** `test_runner_service.py`'s lease round trip, a chunk seeded on the mock hub through
  `/_seed/chunk`, claimed, worked by `mock-claude-code`, and completed, read back from the mock hub's `/_captured` and
  the runner's `GET /runners/{id}/api/leases`.
- **One lever-driven failure.** The unreachable-hub case: the mock hub's `unreachable` lever armed, a completion
  buffered, `GET /runners/{id}/api/runner`'s `hub.buffer_depth` read above zero, the lever cleared, and the buffer seen
  to drain.

Both run on a timed host with `BZ_RUNNER_TICK_SECONDS` small, since nothing steps yet, and lay out the host with
`blizzard-runner init` against the mock hub. The spike lives on a branch and is never merged.

## What it measures and records

The spike records, in this epic's plan, on the CI runner's shape:

- The cost of starting a host, from `init` to `GET /api/health` answering, and of the fixture mint beside it.
- The two tests' wall time on a timed host, and how much of it is waiting for the next tick, which bounds what stepping
  saves.
- Every point where a test needed a value no surface gave it, or wrote a file no contract describes. These become
  [reads.md](./reads.md)'s and [layout.md](./layout.md)'s worklist, and any item the spike finds that this spec does not
  list is added here before building starts.

If a host costs so much that porting the runner's roughly ninety tests at a host each would push the tier past the
gate's slowest job, the plan's fallback, a minted fixture world reused per worker, is built with the harness rather than
after it.
