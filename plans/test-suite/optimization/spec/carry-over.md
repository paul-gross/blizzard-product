# Carry-over

The epic changes how the gate runs and how long some tests take, never what any test asserts. Nothing an installation
holds moves. Every change leaves `master` green and every required check's name unchanged.

## The order of landings

The four work areas are independent of one another and of the tier epics, save where named.

1. **The durations recorder and its artifacts** ([pipeline.md](./pipeline.md#recording-durations)). It changes no
   shard's membership. The outlier measurement can be taken from its first runs.
2. **The committed table and the split**, once ten `push` runs on `master` carry the artifacts, with
   `test_shard_split.py` and the weekly refresh. The change records each shard's time before and after, from the runs
   either side of it.
3. **The amd64-only dev image**, independent of everything ([pipeline.md](./pipeline.md#the-dev-image)).
4. **The board's build on every pull request** and the `web-dist` artifact
   ([pr-gate.md](./pr-gate.md#the-boards-build-on-every-pull-request)), independent of everything.
5. **The UI job's move** onto the shared build, after `epic:test-suite/fleet-tier` lands the job and after 4, once its
   runtime fits ([pr-gate.md](./pr-gate.md#the-ui-job)).
6. **The three waits**, in one change per file ([waits.md](./waits.md)), the rule's extension landing in
   `blizzard-context` with the last.
7. **The outliers**, one change per file the measurement names ([outliers.md](./outliers.md)), after 1.

## What the gate is measured against

The baseline in [the plan](../index.md#baseline) is the measure, taken again with the durations recorder after 2, 3, and
7, each change recording the job it shortened, before and after, in its description. The epic is done when its work
areas have landed and the gate's critical path, the slowest required job on a `push` to `master`, is recorded against
that baseline, whatever it came to.
