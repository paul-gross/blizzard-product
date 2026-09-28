# `blizzard runner models` command contract

## Invocation

```text
blizzard runner models [--dir DIR] [--runner-url URL] [--graph NAME | --chunk CHUNK_ID] [--from-file] [--json]
```

The command is a new verb on the runner's click group, registered beside `status`, whose `--dir` and `--runner-url`
options it shares with the same meaning. `--graph` and `--chunk` are mutually exclusive. It writes nothing: not the
configuration, not the runner store, and not the hub.

## Where each part comes from

**The runner's layers.** When the daemon is reachable — found the way `runner status` finds it — the daemon builds the
report in-process through a new read on its local API, against the configuration, bindings, and harness health it loaded
at startup. Those are what its next spawn will use. If `blizzard-runner.toml` on disk no longer matches what the daemon
loaded, the report says so in its header and in `runner.config_changed_on_disk`. With `--from-file`, or with no daemon
running, the command loads the file through `RunnerConfig.load` and builds the bindings the way `runner tick` does, then
builds the same report in the CLI process. It runs no selftest, so every configured harness shows as `not-probed`.
Selection then treats every configured harness as available and says so. One report builder serves both callers; the
daemon's read and the CLI path differ only in where the configuration and health come from.

**The pools.** The pools come from the hub read in [resolution.md](./resolution.md#the-hub-read), using the hub URL and
runner token already in the runner's configuration:

| Invocation     | Pools reported                                                                        |
| -------------- | ------------------------------------------------------------------------------------- |
| no flag        | Every effective graph's declared sessions, graphs by name, sessions in authored order |
| `--graph NAME` | That graph's declared sessions                                                        |
| `--chunk ID`   | The chunk's pinned graph, merged with the chunk's defaults, plus any bare lineage     |

**What is already running.** Only with `--chunk`, the pool heads this runner holds for that chunk, read from its own
store.

A hub that cannot be reached, or one too old to serve the read, costs the report its pools and nothing else. The pools
section is replaced by one line naming which of the two happened, and the runner's tables still print.

## Which rows appear

- **Harnesses.** Every harness enabled in the configuration, including an unavailable one, with its version, its state
  (`available`, `unavailable`, `unhealthy`, or `not-probed`), whether it is the runner's default, and what it runs when
  nothing resolves.
- **Tiers.** The union of the well-known tiers (`blizzard:frontier`, `blizzard:advanced`, and `blizzard:basic`), every
  tier keyed in any of this runner's model alias tables, and every tier authored in a reported pool. There is one row
  per tier and harness.
- **Efforts.** The union of `low`, `medium`, `high`, and `max`, every key in any of this runner's effort alias tables,
  and every effort authored in a reported pool. There is one row per effort and harness.
- **Pools.** One block per session. Each block holds the three effective fields with their sources, then one line per
  candidate harness in set order with its outcome, model, and effort, then the recorded pool head where one exists.

## The terminal rendering

The sections appear in this order: a header naming the runner, config path, source, and hub scope, then `HARNESSES`,
`TIERS`, `EFFORT`, and `SESSION POOLS`. Output is plain `click.echo` with fixed-width columns, and no color is required
to read it. That matches the rest of the runner CLI. The tables fit in 120 columns, and a long model name widens its
column rather than being truncated. A missing value is rendered as `—`. Sources are rendered as words: `runner alias`,
`adapter default`, `chunk default`.

A line beginning `!` follows the row it concerns whenever a finding changes what runs. There are four such findings: a
tier a harness cannot map, an effort that is dropped, a recorded pool head that differs from the next fresh mint, and a
daemon running configuration older than the file. Nothing else earns a `!`. The
[worked sample](../../artifacts/model-resolution.txt) is the reference layout.

## The JSON form

`--json` prints one JSON document to stdout and nothing else. The flag follows the `--json` option of the hub CLI's
`HubCommand`; it is the first `--json` on the runner CLI. The document carries every layer's value beside the effective
one, never the result alone:

| Key         | Holds                                                                                                                                                                                                                      |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `schema`    | The document version, `1`                                                                                                                                                                                                  |
| `runner`    | Name, config path, `source` (`daemon` or `file`), when the daemon loaded its config, and `config_changed_on_disk`                                                                                                          |
| `hub`       | URL and `status`: `reachable`, `unreachable`, `unsupported`, or `not-asked`                                                                                                                                                |
| `scope`     | The graph name, graph id, and chunk id the pools were asked for, each null when not named                                                                                                                                  |
| `harnesses` | One entry per configured harness: id, version, state, default, and fallback model with its source                                                                                                                          |
| `tiers`     | Per tier, per harness: `adapter_default`, `runner_alias`, the resolved `model`, and `source`                                                                                                                               |
| `efforts`   | Per authored effort, per harness: `runner_alias`, the value that reaches the harness, its `flag`, and `source`                                                                                                             |
| `pools`     | Per session: each field's `graph`, `chunk`, effective `value`, and `source`; `selection_basis` (`acceptable-set` or `runner-default`); the candidates, each with its outcome and skip `reason`; `selected`; and `recorded` |
| `notes`     | The `!` findings as data: a `kind` (`unmapped-tier`, `dropped-effort`, `recorded-differs`, `stale-daemon-config`) and what it names                                                                                        |

Every source is a term from the [provenance vocabulary](./resolution.md#provenance-vocabulary), and every outcome is a
term from [harness selection](./resolution.md#harness-selection). Adding a key keeps `schema` at `1`, but renaming or
removing one raises it. The [sample JSON](../../artifacts/model-resolution.json) shows the whole shape on the same
configuration as the terminal sample.

## Exit status

| Code | When                                                                                                                                                                 |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0`  | The report printed. Findings do not change this: the command reports, it does not judge                                                                              |
| `1`  | The configuration failed to load, with the same message `runner host` gives                                                                                          |
| `2`  | A usage error, including `--graph` with `--chunk`                                                                                                                    |
| `3`  | `--graph` or `--chunk` was given and its pools could not be reported: the hub is unreachable or unsupported, or the name is unknown. The runner's tables still print |
