# Carry-over

The epic changes where the hub's service tests run and adds small surfaces: the tags on background passes' lines, sweeps
on request, the stream's join from now, the login throttle's keys, and the IdP's keyed sign-in. A hub whose file names
no sweep mode runs every sweep on its timer, as today. Nothing an installation holds moves.

## The order of landings

Each change leaves `master` green and the required check intact. Every port waits on `epic:test-suite/foundation`'s
package having landed.

1. **The surfaces**, each independent save one: the pass tags, the sweeps on request after the tags, the stream's
   `from=latest` in the hub and the runner, the login throttle's keys, and keyed sign-in in `blizzard-mock`
   ([surfaces.md](./surfaces.md)).
2. **The harness's additions** ([harness.md](./harness.md)) and the barrier ([waits.md](./waits.md#the-barrier)), with
   the profiles, every one of them setting every sweep `stepped`, after 1's sweep, stream, and IdP changes. The
   superuser sign-in, people, and the `auth` profile's throttle setting land together, proven by the harness's own spec
   signing several people in concurrently on one shared `auth` hub and each coming back as itself. The mock runner's
   client ([harness.md](./harness.md#the-mock-runners-client)) is independent of 1, and lands before the first port that
   drives a mock runner.
3. **The helpers that move first** ([below](#helpers-that-move-first)).
4. **The ports**, one change per file or per split file, in any order after 2 and 3: the HTTP group first, since it
   needs only the default and enforcement profiles, then the `auth` group. Each port deletes its Python original, or the
   tests it took from a split file, places each test it moves by the read rules, and moves its citations in the same
   change.
5. **`tenant_teardown` to the component tier**, independent of everything.

Once the HTTP group has ported, the foundation's hubs-per-worker measurement
([its carry-over](../../foundation/spec/carry-over.md#the-measurements)) is taken again over the tier as it then stands,
since the foundation took it over `fresh-tenant` tests alone, and `WORKERS_PER_HUB` moves if the new numbers ask. The
same measurement counts the `fresh-hub` tests the ports placed and their cost, and if they push the tier past the gate's
slowest job, the placements that cost the most are revisited.

## Helpers that move first

A port deletes the file it moves, so whatever a file that outlives it imports from that file, in another tier or in a
service file not yet ported, moves to a support module in a change of its own, before any port. With every shared helper
out of the ported files, no port waits on another, and the ports land in any order:

| Helper                                                                                    | Imported by                                                                                                    | Moves to                                      |
| ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| `invite`, in `test_auth_login_service.py`                                                 | `tests/e2e/test_login_session_e2e.py`, `test_cli_login`, `test_event_stream_revocation`, `test_idp_federation` | `tests/support.py`, beside `seed_invitation`  |
| `_oauth_hub` and `_SECRET_ENV`, in `test_auth_login_service.py`                           | `test_cli_login`, `test_event_stream_revocation`, `test_idp_federation`                                        | `tests/service/support.py`, beside `stub_idp` |
| `_graph_yaml`, in `test_hub_service.py`                                                   | `tests/e2e/test_event_log_e2e.py`, `test_event_log`                                                            | `tests/support.py`, as `graph_yaml`           |
| `_stack` and `_ingest`, in `test_hub_service.py`                                          | `test_event_log`, `test_runner_served_projects`, `test_runner_declining`, `test_runner_auth`                   | `tests/service/support.py`                    |
| `Garden`, `garden_stack`, `deliver`, `seed`, `add_op`, in `test_finding_exits_service.py` | `test_routine_baselines`                                                                                       | `tests/service/support.py`                    |

The service files in the second column are named without their `_service.py` suffix. `blizzard-context`'s
`e2e-scenarios/auth.md` cites `invite` in its old home, and the citation moves with it.

`tests/service/support.py` stays, since the pytest half and e2e still use `stub_idp`, `require_stub_idp`, `mock_runner`,
and `poll_until` from it. Its `sse_tap` and `SseTap` stay while the runner tier's stream tests still use them.
`seed_invitation` stays while e2e calls it, and the rule against seeding through the store is the service tier's, not
e2e's.

## Citations

Each port re-points every citation of the original to its spec, under `bzh:port-moves-citations`: the file names under
`blizzard-context`'s `verification/` and `architecture/`, the bare function names, which keep resolving because the
titles keep them, and the original's entries in `scripts/prose-density-baseline.json`, which are dropped for a file that
is deleted and measured again for a split file that stays with its remaining tests. The registry check confirms each
change: a citation of a deleted file fails it.

## Done

Every file in [roster.md](./roster.md)'s *What moves* runs in `tests/service/hub/` under the profile and level it was
placed at, the files in *What stays in Python* still run in the pytest half, `tests/test_tenant_teardown.py` runs in the
component tier, and the tier's CI time before and after is recorded in the plan.
