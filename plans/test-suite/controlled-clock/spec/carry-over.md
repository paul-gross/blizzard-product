# Carry-over

Every landing is additive. A daemon started without the fence behaves exactly as it does today, and nothing an
installation holds moves.

## The order of landings

1. **The federation token's `iat` on the injected clock**
   ([clock.md](./clock.md#the-token-check-that-must-reach-the-clock)). It stands on its own and must land before
   anything moves a clock.
2. **`ControlledClock` and the fence** in `src/blizzard/foundation/clock.py`, with their component tests. Nothing calls
   them yet.
3. **The hub's composition root and its clock routes** ([clock.md](./clock.md#the-composition-roots),
   [routes.md](./routes.md)), after 2. A controlled hub also runs its flow driver stepped and mounts the flow-step route
   behind the clock fence ([routes.md](./routes.md#the-flow-drivers-step)). The hub's existing flow-step control keeps
   working beside it until 7.
4. **The runner host's composition root and its clock routes**, after 1 and 2, independent of 3.
5. **The mock fleet's clocks and `/_clock` routes**, in `blizzard-mock`, independent of everything before it.
6. **`docs/versioning.md`'s sentence on fenced surfaces** ([routes.md](./routes.md#not-a-released-surface)), with 3, the
   first change that mounts a fenced route.
7. **The clock fixture**, after 3, 4, and 5: the `clockMode` option and its refusals, the environment, alignment, moves,
   session refresh, the browser, and `vendor-usage.ts` reading `clock.now()` ([harness.md](./harness.md)). A harness
   check spec proves alignment, a move across hub, host, and forge, and the refusal at a shared isolation level. Until
   8, a hub refuses to boot with both the controlled clock and the flow-step control permitted.
8. **The flow-step control's retirement**, after 7 ([harness.md](./harness.md#the-flow-step-control-it-retires)),
   independent of the ports.
9. **The ports**, after 7 ([ports.md](./ports.md)), in three changes: the fleet specs that wait out real windows, the
   budget lever with the subscription floor, and the release with the utilization lever.
10. **The check against the machine's clock and the rulebook's rewrite**, after the last port
    ([below](#citations-and-the-rulebook)).

## Citations and the rulebook

In `blizzard-context`:

- `bzh:injected-clock` (`architecture/crash-correctness.md`) names the controlled clock, its fence, and the rule that
  only domain time moves under it, in the change that lands 2.
- `bzh:service-waits-on-the-system` (`verification/blizzard/service-tests.md`) loses its exception for wall-clock
  expiry. A wait whose subject is a window ageing out no longer polls with a deadline covering the window. It advances
  the stack's clock past the window and steps. The rule gains the controlled clock's own Do, and its Detect names the
  new lint. This lands in 10, since the rule must not refuse a spec the suite still holds.
- `verification/blizzard/service-tests.md` gains how a test asks for the clock and who may, beside its account of the
  stepped host, in the change that lands 7.

Each port re-points nothing, since a port keeps its file and title. A spec whose case is new, such as the subscription
floor's reset, is cited where its subject already is, in the same change.

## Open questions

- **A worker's own timeline.** The mock harness stamps its transcript lines from the machine's clock
  (`blizzard_mock/harness/facades/_transcript.py`), as a real coding harness does. On a stack advanced by an hour, every
  transcript line reads an hour behind the runner. No rule known today compares a transcript stamp with the runner's
  `now`, so the epic leaves workers on real time. Whether the mock harness should also take the stack's clock, through
  an environment variable at spawn, is open.
- **Ties under a frozen clock.** Every write a pass makes between two moves shares one instant. A read that orders by
  instant alone, with no tiebreak, would order them arbitrarily, and the controlled clock would surface that as a
  failing test. The plan treats such a failure as a product defect to fix in the read, since a real clock also ties at
  its resolution, rather than making the controlled clock tick on its own. Whether that stance holds once the first tie
  appears is open.
- **The CLI's clock.** `blizzard` the CLI judges a window such as `--since 1h` against the machine's clock
  (`CliCollaborators`, `src/blizzard/cli/collaborators.py`). A test driving the CLI against an advanced hub passes
  explicit instants. Whether the CLI should read the hub's clock through the route when one answers is open.
