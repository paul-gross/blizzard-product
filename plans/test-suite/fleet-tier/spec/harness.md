# The harness

What the fleet and UI projects add to the harness the earlier epics built. The runner tier's `runnerHost`, its
`runnerEnv`, `stepUntil`, and the `service-fleet[-<profile>]` projects `harness/projects.ts` already generates are used
as they stand.

## Hub profiles

A profile is a hub configuration file at `tests/service/hub/<name>/profile.toml`, merged into the hub file; a fleet
subfolder of that name selects it, and `strayProfiles` keeps profile files out of `service/fleet/`. Shared hubs start
with no environment of their own, and `hubEnv` is refused below `fresh-hub`.

| Profile   | What it sets                                                                                                                                     | Who runs on it                                                                                  |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| default   | The shared hub as the hub tier configures it, every sweep stepped                                                                                | Most ports, at `fresh-tenant`                                                                   |
| `actions` | The default, and the environment `BLIZZARD_TEST_ACTIONS=1` and `BLIZZARD_CRASH_FENCE=1`, the fence that admits `test.answer` and `test.sequence` | event log, gardening dialog, hub actions, ideation, and any port whose graph uses a test action |
| `auth`    | The hub tier's sign-in profile                                                                                                                   | login, federation                                                                               |

The fence is environment-only by design, so no installation's configuration can turn it on. The harness gains an
environment owned by one named profile: `actions/profile.toml` sits under `tests/service/hub/`, and the harness starts
that profile's hubs, and only those, with the fence set, as `hostHub` already sets `BLIZZARD_TEST_IDP_SECRET` for the
hubs that need it. A harness check proves the fence reaches no other profile's hub, and the daemon environment
contract's `unset` list gains `BLIZZARD_CRASH_POINT`, so a developer's stray crash point cannot kill a shared hub the
fence has armed. The refusal case in `test_hub_actions_e2e.py` owns a hub without the fence.

The egress files take no profile: each is `fresh-hub`, and passes its egress section and `BZ_EGRESS_PATH_KEY` through
`hubConfig` and `hubEnv`, so no shared hub stands up for them.

## Trace companions

Hub tracing and its export endpoint are configured per hub, so a trace assertion owns its hub. Each file in the roster
marked with a trace companion splits in its port: the core cases run at their level on a shared hub, untraced, and a
companion test at `fresh-hub` runs the same scenario once with hub and runner tracing pointed at the sink, and asserts
only on the trace. The companion keeps the original's function name with a `_traced` suffix, so its citation moves
mechanically.

## The sink and the collector

`harness/otlp-sink.ts` decodes a span's trace id, span id, name, and attributes. It grows to decode the rest: its
parent, status, start and end times, events, and links. The shape checks the Python runs against
`contracts/traces/dictionary.json`, in `tests/e2e/fleet_traces.py`, are ported to a TypeScript module beside the sink
that reads the dictionary itself, since `harness/trace-ids.ts` reads only its trace-id entry.

Tail sampling's subject is the documented collector configuration, which only a real collector runs. Those two cases
start `otelcol-contrib` beside their hub and read what it forwards to the sink. The binary is pinned as a `tools` entry
of `mise.toml`'s `playwright` task, at the version `collector-config-check` pins, so the Playwright run has it on `PATH`
as the e2e task's run does today; a run without it fails rather than skips. The CI job that runs `service-fleet` gains
the collector's install and the cache the e2e job keeps today.

## The runner host

The harness gains reads of a bare origin beside `originHead`: a repository's tree at a ref, its commit history, by path
if asked, and a file's contents, so no test needs the mock forge to serve them. Fleet specs may not touch the file
system themselves, so the winter trap `test_basic_workspace.py` puts on the host's `PATH` is written by a fixture, and
the egress files' exported Parquet is read by a fixture over a Node DuckDB dependency the package gains. Two board ports
run a host inside a UI project, against the runner's served board.

## The UI project

`tests/ui/` becomes the package's one project with a browser installed, `ui[-<profile>]`, crossed with the same hub
profiles, against the board the hub serves from `src/blizzard/static/{hub,runner}`. A `ui` task in `mise.toml` depends
on `web-build`, so a stale bundle cannot pass for the current one. The viewport sizes the e2e conftest pins move to a UI
fixture, and blizzard-context's tier rules cite it in their place.

The UI tests are gated from their first port. blizzard-context's verifiability matrix gains a UI method with its command
page, and `upper-tiers.yml` gains a `ui` job beside the e2e job, under `full_suite`, that builds the board itself, as
the e2e job does today, installs Chromium with `npx playwright install --with-deps chromium`, and runs `mise run ui`. It
lands as a required check, `upper-tiers / ui (blizzard:ui)`, before the first browser scenario leaves e2e, so every
browser scenario is gated on every pull request throughout. `epic:test-suite/optimization` later moves the job beside
the frontend build and onto its shared artifact.
