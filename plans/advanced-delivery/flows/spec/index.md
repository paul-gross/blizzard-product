# Flows specifications

The technical contracts behind `epic:advanced-delivery/flows`. Product intent begins at the [epic plan](../index.md);
implementation enters at the concern below. What a hub node's action does, how its outcome routes, and pending and its
poll are owned by the [actions epic](../../actions/spec/index.md) and `domain/artifacts/delivery.md`; landing only where
a chunk's project links is owned by `initiative:projects`'
[delivery contract](../../../../delivered/projects/hub/spec/delivery.md). This spec changes when a hub step runs and
what it waits behind; it changes nothing an action does once it runs.

This spec assumes only the [actions](../../actions/spec/index.md) epic has landed: a hub step is one of the hub's
actions, taken a bounded pass at a time on the `ActionLoop`'s dedicated thread (`hub/delivery/action_loop.py`), handing
every effect it makes to an `IEffectSink` rather than writing it, and declaring in its catalogue entry,
`ActionSpec.lands` (`hub/domain/actions/catalogue.py`), whether it holds the branches its chunk lands on. It builds on
what is already true of the code: a repository carries an id (`repo_<ulid>`), a slug it may change, and a name, and
every reference holds the id, so a resource names its repository by id and every surface joins the current slug and name
when it reads; and each of a chunk's commit pointers resolves to its own repository record, with its own base branch,
forge address, owner, and credential.

Paths are relative to `blizzard/src/blizzard/`.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`:

- `domain/artifacts/delivery.md` replaces its fleet-wide serialization paragraph with resources and flows, and corrects
  the scope the code already has: one slot per tenant, never one per fleet. Its "only a live, promoted chunk is driven"
  rule stands, now as the condition the driver's reconcile submits on ([verdict.md](./verdict.md#submitting)).
- `domain/work/statuses.md` says a `delivering` chunk is submitted to its flows or taking its turn in them.
- `domain/operations.md` gains the event kinds in [record.md](./record.md).
- `architecture/crash-correctness/hub.md` replaces §The boot release of live hub-execution slots with the boot recovery
  of open turns ([carry-over.md](./carry-over.md#at-boot)), registers the `flow.` crash-point family
  ([turns.md](./turns.md#crash-points)), and names `FlowDriver` among the hub's lanes in
  `architecture/crash-correctness/lanes.md`.
- `architecture/system-shape/exclusive-writes.md`'s single-process known debt replaces the boot slot release with the
  boot turn recovery and the driver's in-memory sweep of open turns ([turns.md](./turns.md#a-turn-that-cannot-end)).
- `architecture/system-shape/tenancy.md`'s list of hub-wide jobs that loop over open tenants replaces "boot slot
  release" with "boot turn recovery".

`verification/blizzard/e2e-scenarios/board.md` gains the flow lanes. Runners are untouched: the runner still calls
hub-advance while its chunk is `delivering` and reads nothing new from the answer.

| Where                            | Read when                                                                                                                                     |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| [turns.md](./turns.md)           | Replacing the slot — what a step holds, a turn, who is next on a branch, the hub's flow driver, pending, a turn that cannot end, crash points |
| [record.md](./record.md)         | Shaping the store — flows, submissions, turns, the append-only flow record, the landed event, invariants                                      |
| [verdict.md](./verdict.md)       | Submitting and answering — the driver's reconcile, how a turn's outcome becomes its verdict, escalation, landing across several repositories  |
| [levers.md](./levers.md)         | Pause, stop, restart, complete, requeue, and detach against a chunk that is submitted or mid-turn                                             |
| [surfaces.md](./surfaces.md)     | Building the API, CLI, live channel, and the board's flow lanes and card positions                                                            |
| [carry-over.md](./carry-over.md) | Writing the migration — retiring the slot, admitting chunks already mid-delivery at the head of their flows, and the proof                    |
