# The harness

What the fleet and UI projects add to the harness the earlier epics built. The runner tier's `runnerHost`, `stepUntil`,
and the `service-fleet[-<profile>]` projects `harness/projects.ts` already generates are used as they stand.

## Hub profiles

A fleet test's hub profile lives in a subfolder of `tests/service/fleet/` named for it, as the hub tier's do:

| Profile   | What it sets                                                                                                                      | Who runs on it                                                                                  |
| --------- | --------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| default   | The shared hub as the hub tier configures it, every sweep stepped                                                                 | Most ports, at `fresh-tenant`                                                                   |
| `actions` | The default, plus `BLIZZARD_TEST_ACTIONS=1` and `BLIZZARD_CRASH_FENCE=1`, the fence that admits `test.answer` and `test.sequence` | event log, gardening dialog, hub actions, ideation, and any port whose graph uses a test action |
| `egress`  | An egress section and `BZ_EGRESS_PATH_KEY`, always `fresh-hub`                                                                    | the two egress files                                                                            |
| `auth`    | The hub tier's sign-in profile                                                                                                    | login, federation                                                                               |

A profile that admits test actions never admits them on a hub other profiles share. The refusal case in
`test_hub_actions_e2e.py` owns a hub without the fence.

## Trace companions

Hub tracing and its export endpoint are configured per hub, so a trace assertion owns its hub. Each file in the roster
marked with a trace companion splits in its port: the core cases run at their level on a shared hub, untraced, and a
companion test at `fresh-hub` runs the same scenario once with hub and runner tracing pointed at the sink, and asserts
only on the trace. The companion keeps the original's function name with a `_traced` suffix, so its citation moves
mechanically.

## The sink and the collector

`harness/otlp-sink.ts` decodes a span's trace id, span id, name, and attributes. It grows to decode the rest of a span:
its parent, status, start and end times, events, and links, so the companions that check backlog-wait arithmetic and
span trees read them from the sink. It keeps reading `contracts/traces/dictionary.json` through `harness/trace-ids.ts`.

Tail sampling's subject is the documented collector configuration, which only a real collector runs. Those two cases
start `otelcol-contrib` beside their hub, pinned by the same version `collector-config-check` pins in `mise.toml`, and
read what it forwards to the sink. The harness finds the binary through `mise which`, and a run without it fails rather
than skips.

## The runner host

`runnerHost` gains an environment the test controls, so `test_basic_workspace.py` can run the host with a `PATH` whose
`winter` is a trap. Two board ports run a host inside a UI project, against the runner's served board.

## The UI project

`tests/ui/` becomes the package's one project with a browser installed: `ui[-<profile>]`, crossed with the same hub
profiles, against the board the hub serves from `src/blizzard/static/{hub,runner}`. Locally the board is built by
`mise run web-build`, which the UI task depends on, so a stale bundle cannot pass for the current one; in CI the job
downloads `gate.yml`'s `web-dist` artifact. `npx playwright install --with-deps chromium` is the job's, and no other
project installs a browser. The viewport sizes the e2e conftest pins move to a UI fixture, and blizzard-context's tier
rules cite it in their place.
