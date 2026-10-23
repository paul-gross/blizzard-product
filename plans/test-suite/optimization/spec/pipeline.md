# The pipeline

## Recording durations

A split by time needs times, and none are kept today. The suite plugin gains a recorder: when `BLIZZARD_DURATIONS_OUT`
names a file, `tests/suite_plugin.py` adds each test's setup, call, and teardown durations from
`pytest_runtest_logreport` and, at session end, writes `{ "<nodeid>": <seconds>, ... }` to that file. Under xdist each
worker writes its own part, `<file>.<worker id>`, and the controller merges them at `pytest_sessionfinish`, so the file
holds every test the run executed exactly once. A skipped test is not recorded, since its time says nothing about its
cost when it runs.

`gate.yml`'s `python-tests` job and `upper-tiers.yml`'s `service-tier` job set `BLIZZARD_DURATIONS_OUT=durations.json`
and upload it as an artifact, `durations-<job>-<shard>`, on every run, with `if: ${{ !cancelled() }}` so a red run's
times are kept too. They are kept for fourteen days.

## The committed table

The split must be the same in every shard, so it cannot read times a shard has not seen. It reads a committed table,
`tests/durations.json`, holding the median duration of each test over the runs it was built from, rounded to a hundredth
of a second, keyed by nodeid and sorted.

`scripts/test-durations.py refresh` builds it: it downloads the `durations-*` artifacts of the last ten successful
`push` runs on `master` with `gh run download`, takes each nodeid's median over the runs that executed it, drops nodeids
no longer collected, and writes the table. `mise run test-durations` runs it. The table is refreshed by an ordinary
change whenever the shards drift apart, and a scheduled workflow, `.github/workflows/durations.yml`, runs `refresh`
weekly and opens a pull request when the table changed by more than five percent of any shard's total, so drift is
noticed without anyone watching for it.

## The split

`shard_of` gives way to a partition computed once per collection. `pytest_collection_modifyitems` loads
`tests/durations.json`, gives every collected test its recorded duration, or the table's median duration when it has
none, so a new test is placed as an ordinary one rather than as free, and assigns shards by longest-processing-time
packing: tests sorted by duration descending, ties broken by nodeid, each placed in the shard whose running total is
smallest, ties to the lowest index. Every shard computes the same partition from the same collection and the same table,
so every test lands in exactly one shard, as today. A test missing from the table lands somewhere each time,
deterministically, and is placed by its real time after the next refresh.

The table is consulted only for the order and the totals; a shard still runs `-n auto` within itself, and xdist
schedules within the shard as it does now. With `BLIZZARD_TEST_SHARD` unset nothing is deselected, so a local
`uv run pytest` is unchanged. A missing or unreadable `tests/durations.json` falls back to `shard_of`'s hash with a
warning, so the gate never fails for want of times.

`tests/test_shard_split.py`, a unit test, holds the partition to its promises over a synthetic collection: every item in
exactly one shard, the same partition from any item order, a missing table falling back to the hash, and the largest
shard's total within the longest single test of the mean.

The service tier's pytest half shards through the same hook, so it is balanced too, from its own durations in the same
table.

## The dev image

`push.yml`'s `dev-image` builds `--platform linux/amd64` only, and drops `docker/setup-qemu-action`. The `edge` and
`sha-<sha>` tags become single-platform images; the hosted hub, the only consumer of `edge`, runs on amd64. Signing and
provenance are unchanged, and `cosign sign --recursive` still signs what is there.

`release.yml` keeps both platforms, QEMU, and a bill of materials per platform, so every release is still published for
arm64. The first arm64 build of any change is therefore the release candidate's, and `docs/ci.md` says so beside the dev
channel's description.
