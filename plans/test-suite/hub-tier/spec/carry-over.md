# Carry-over

The epic changes where the hub's service tests run and adds three small surfaces, the tags on background passes' lines,
the stream's join from now, and the IdP's keyed sign-in. Nothing an installation holds moves.

## The order of landings

Each change leaves `master` green and the required check intact. Every port waits on `epic:test-suite/foundation`'s
package having landed.

1. **The classification.** A table in [roster.md](./roster.md)'s shape over every test, not every file, with the level
   the read rules place each at, written into the plan beside the roster before the first port, with the count of
   `fresh-hub` tests and their cost at about ten seconds each. If that cost would push the tier past the gate's slowest
   job, the classification is revisited before any port lands.
2. **The surfaces**, each independent: the pass tags, the stream's `from=latest` in the hub and the runner, and keyed
   sign-in in `blizzard-mock` ([surfaces.md](./surfaces.md)).
3. **The harness's additions** ([harness.md](./harness.md)) and the barrier ([waits.md](./waits.md#the-barrier)), with
   the profiles, after 2's stream and IdP changes. The superuser sign-in, people, and the `auth` profile's throttle
   setting land together, proven by the harness's own spec signing several people in on one shared `auth` hub in quick
   succession and each coming back as itself.
4. **The helpers that move first** ([below](#helpers-that-move-first)).
5. **The ports**, one change per file or per split file, in any order after 3 and 4: the HTTP group first, since it
   needs only the default and enforcement profiles, then the `annotation` files, then the `auth` group. Each port
   deletes its Python original, or the tests it took from a split file, and moves its citations in the same change.
6. **`tenant_teardown` to the component tier**, independent of everything.

## Helpers that move first

A port deletes the file it moves, so whatever another tier imports from that file moves to a support module in a change
of its own, before the port:

| Helper                                    | Imported by                           | Moves to                                     |
| ----------------------------------------- | ------------------------------------- | -------------------------------------------- |
| `invite`, in `test_auth_login_service.py` | `tests/e2e/test_login_session_e2e.py` | `tests/support.py`, beside `seed_invitation` |
| `_graph_yaml`, in `test_hub_service.py`   | `tests/e2e/test_event_log_e2e.py`     | `tests/support.py`, as `graph_yaml`          |

`tests/service/support.py` stays, since the pytest half and e2e still use `stub_idp`, `require_stub_idp`, `mock_runner`,
and `poll_until` from it. Its `sse_tap` and `SseTap` stay while the runner tier's stream tests still use them.
`seed_invitation` stays while e2e calls it, and the rule against seeding through the store is the service tier's, not
e2e's.

## Citations

Each port re-points every citation of the original to its spec, under `bzh:port-moves-citations`: the file names under
`blizzard-context`'s `verification/` and `architecture/`, the bare function names, which keep resolving because the
titles keep them, and the original's entries in `scripts/prose-density-baseline.json`, which are dropped. The registry
check confirms each change: a citation of a deleted file fails it.

## Done

Every file in [roster.md](./roster.md)'s *What moves* runs in `tests/service/hub/` under the profile and level it was
placed at, the files in *What stays in Python* still run in the pytest half, `tests/test_tenant_teardown.py` runs in the
component tier, and the tier's CI time before and after is recorded in the plan.
