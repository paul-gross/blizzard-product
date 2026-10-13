# Flows specifications

The technical contracts behind the flows slice of `epic:advanced-delivery`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. What a hub node's action does, how its outcome
routes, and pending and its poll are owned by the [actions slice](../../actions/spec/index.md) and
`domain/artifacts/delivery.md`; landing only where a chunk's project links is owned by `epic:projects`'
[delivery contract](../../../../delivered/projects/hub/spec/delivery.md). This spec changes when a hub step runs and
what it waits behind; it changes nothing an action does once it runs.

This spec assumes the [actions](../../actions/spec/index.md) and [forges](../../forges/spec/index.md) slices have
landed: a hub step is one of the hub's actions, taken a short pass at a time and saying whether it lands, and landing
speaks the forge seam. It builds on what is already true of the code: a repository carries an id (`repo_<ulid>`), a slug
it may change, and a name, and every reference holds the id, so a resource names its repository by id and every surface
joins the current slug and name when it reads; and each of a chunk's commit pointers resolves to its own repository
record, with its own base branch, forge, owner, and credential.

Paths are relative to `blizzard/src/blizzard/`.

The slice reaches beyond `blizzard`'s source. In `blizzard-context`, `domain/artifacts/delivery.md` replaces its
fleet-wide serialization paragraph with resources and flows, and corrects the scope the code already has: one slot per
tenant, never one per fleet. `domain/work/statuses.md` says a `delivering` chunk is submitted to its flows or taking its
turn in them. `domain/operations.md` gains the event kinds in [record.md](./record.md).
`verification/blizzard/e2e-scenarios/board.md` gains the flow lanes. Runners are untouched: the runner still calls
hub-advance while its chunk is `delivering` and reads nothing new from the answer.

| Where                            | Read when                                                                                                                      |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| [turns.md](./turns.md)           | Replacing the slot — what a step holds, a turn, who is next on a branch, the hub's flow driver, pending, the turn's time bound |
| [record.md](./record.md)         | Shaping the store — flows, submissions, turns, the append-only flow record, the landed event, invariants                       |
| [verdict.md](./verdict.md)       | Submitting and answering — how a chunk submits, how a turn's outcome becomes its verdict, landing across several repositories  |
| [levers.md](./levers.md)         | Pause, stop, restart, complete, and detach against a chunk that is submitted or mid-turn                                       |
| [surfaces.md](./surfaces.md)     | Building the API, CLI, live channel, and the board's flow lanes and card positions                                             |
| [carry-over.md](./carry-over.md) | Writing the migration — retiring the slot, admitting chunks already mid-delivery at the head of their flows, and the proof     |
