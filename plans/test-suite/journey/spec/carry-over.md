# Carry-over

The epic adds two routes and a client mode to a hidden CLI, all additive, and changes nothing an installation holds. The
offline check, the crash sweep's in-process checker, and every other caller of `Invariants` behave as they do today.

## The order of landings

1. **The parity repair** ([parity.md](./parity.md#the-repair)), independent of everything.
2. **The ref resolver, the parity job, and the nightly run**, after 1, the job optional at first, then required by a
   change of its own, with blizzard-context's mock command and companion pages ([parity.md](./parity.md#the-jobs)).
3. **The checker's split** ([check.md](./check.md#the-checkers-split)): injected connections and probe, the direct
   callers rewired, and the amendment of `bzh:hub-scoped-reads-confined`. It changes no check.
4. **The Postgres method** in blizzard-context's matrix ([check.md](./check.md#one-read)), any time before 5.
5. **The two routes and their one read**, after 3 and 4: the scoped-connections form over an open connection, the
   single-run refusal, component tests, the Postgres proof, the OpenAPI documents, the generated clients, the change
   note, and blizzard-context's statement of the checker with the crash sweep's page corrected
   ([check.md](./check.md#two-routes), [check.md](./check.md#the-rule)).
6. **The CLI's client mode**, after 5 ([check.md](./check.md#the-clis-two-modes)).
7. **The port** ([port.md](./port.md)), after 6 and the fleet tier's bare-origin reads, deleting the Python rehearsal in
   the same change and re-pointing `mise run journey`.

## Citations and the rulebook

The port moves the rehearsal's citations under `bzh:port-moves-citations`, and its command page in blizzard-context
names the Playwright project, keeping the command text the registry checks. The crash sweep's command page is corrected
in 5, and the mock's command and companion pages in 2.
