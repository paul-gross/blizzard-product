# Optimization specifications

**Provisional.** This spec is drawn against the gate as it runs before the service tier's move, and that gate will not
be the one this epic finally works on: the foundation, hub, runner, and fleet tiers each reshape the jobs it measures.
It is re-grounded against the gate those four leave behind, and its baseline taken again, before anything is built from
it.

The technical contracts behind `epic:test-suite/optimization`. Product intent and the baseline begin at the
[epic plan](../index.md); implementation enters at the concern below. The service tier's own cost is the tier epics',
and nothing here touches it, save that the duration-balanced split serves the service tier's pytest half as well,
through the same hook.

It builds on the gate as it runs today. `pr.yml` and `push.yml` each call `gate.yml` and `upper-tiers.yml` side by side,
so every check starts at once. `gate.yml`'s `python-tests` job runs `uv run pytest -n auto` in four shards,
`BLIZZARD_TEST_SHARD=<shard>/4`, and `tests/suite_plugin.py`'s `pytest_collection_modifyitems` keeps a test in shard
`zlib.crc32(nodeid) % count + 1` (`shard_of`), so the split is even by count and blind to time; `python-tests-result`
reports them under `pytest (unit + component)`. Nothing records a test's duration anywhere: no `--durations`, no JUnit
report, no artifact. The `frontend` job runs `npm run test`, which is `ng test hub && ng test runner && ng test fleet`,
three projects in sequence, and builds the hub only inside `bundle-check` and Storybook inside `story-render`; the full
`npm run build` runs first in `push.yml`'s `dev-build`, through `scripts/build-wheel.sh`. `push.yml`'s `dev-image`
builds `linux/amd64,linux/arm64` under QEMU on every push.

Paths are relative to the `blizzard` repo root unless a leaf says otherwise. The epic changes nothing outside
`blizzard`: the hosted hub already runs on amd64 (`blizzard-infra`'s `terraform/application/config/prod.tfvars`, a
`t3a.small` on an amd64 AMI), so the image's prerequisite is met.

| Where                            | Read when                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| [pipeline.md](./pipeline.md)     | Recording test durations, splitting shards by them, or building the dev image for one platform                 |
| [pr-gate.md](./pr-gate.md)       | Building the board ahead of time on every pull request, or adding the UI job                                   |
| [waits.md](./waits.md)           | Replacing a fixed wait outside the service tier with one that ends when the thing has happened                 |
| [outliers.md](./outliers.md)     | Measuring and shortening the slow unit and component files, `test_runner_harness_opencode_diagnostic.py` first |
| [carry-over.md](./carry-over.md) | Landing the epic — the order of changes, what each records, and what the gate is measured against              |
