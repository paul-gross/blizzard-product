# Carry-over

The epic adds two routes and a client mode to a hidden CLI, all additive, and changes nothing an installation holds. The
offline check, the crash sweep's in-process checker, and every other caller of `Invariants` behave as they do today.

## The order of landings

1. **The parity repair** ([parity.md](./parity.md#the-repair)), independent of everything.
2. **The ref resolver and the parity job**, after 1, optional at first, then required by a change of its own, with the
   nightly run beside it ([parity.md](./parity.md#the-jobs)).
3. **The checker's split** ([check.md](./check.md#the-checkers-split)), changing no check.
4. **The two routes and their one read**, after 3, with component tests, the Postgres proof, the OpenAPI documents, the
   generated clients, and the change note ([check.md](./check.md#two-routes)).
5. **The CLI's client mode**, after 4 ([check.md](./check.md#the-clis-two-modes)), with blizzard-context's statement of
   the checker in the same change ([check.md](./check.md#the-rule)).
6. **The daemon-only kill** in the harness ([port.md](./port.md#topology)), any time.
7. **The port** ([port.md](./port.md)), after 5, 6, and the fleet tier's `actions` profile, deleting `tests/journey/` in
   the same change.

## Citations and the rulebook

The port moves the rehearsal's citations under `bzh:port-moves-citations`, and its command page in blizzard-context
names the Playwright project. The crash sweep's command page is corrected in 5, and the mock's command and companion
pages in 2.
