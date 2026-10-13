# Trains specifications

The technical contracts behind the trains slice of `epic:advanced-delivery`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. The flow, its submissions, its turns, its record,
and how a verdict reaches a chunk are owned by the [flows slice](../../flows/spec/index.md); the landing seam and its
capabilities by the [forges slice](../../forges/spec/index.md); and the repository's policy, the order its repositories
land in, the reserved waits, and the rate budget by the [policies slice](../../policies/spec/index.md). This spec
teaches a flow to land many submissions at once, and to hand its line to a forge that keeps one of its own.

This spec assumes the [actions](../../actions/spec/index.md), [forges](../../forges/spec/index.md),
[flows](../../flows/spec/index.md), and [policies](../../policies/spec/index.md) slices have landed: `land` runs each
repository's policy a pass at a time; every forge is spoken through `IForgeLanding`, including `merge_into` and the
compare-and-swap `advance_ref`; and each branch's flow gives its submissions turns, one at a time, in arrival order.

Paths are relative to `blizzard/src/blizzard/`.

The slice reaches beyond `blizzard`'s source. In `blizzard-context`, `domain/artifacts/delivery.md` says a branch's flow
may land a batch, that the revision landed is the revision proven, and that a chunk is ejected rather than rebased by an
agent when it cannot join; `domain/operations.md` gains the event kinds in [record.md](./record.md);
`verification/blizzard/e2e-scenarios/delivery-policy.md` gains the batching and queue scenarios, and
`verification/blizzard/e2e-scenarios/board.md` the batch on a lane. `docs/deployment/` gains a page on batching —
including the CI a repository must run on its train branches — and on handing the line to a forge's queue.
`blizzard-mock`'s forge grows what the proofs need ([carry-over.md](./carry-over.md#the-mock)).

| Where                                | Read when                                                                                                             |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| [capabilities.md](./capabilities.md) | Narrowing a forge's capabilities to one repository, probing what its forge has switched on, and the batching settings |
| [batches.md](./batches.md)           | Building the flow's own train — readiness, forming, building, proving, landing, splitting, and ejecting               |
| [queue.md](./queue.md)               | Building the `forge-queue` policy — joining GitHub's merge queue or GitLab's merge train, and its fallback            |
| [record.md](./record.md)             | Shaping the batch tables, the train turn, the new facts and events, and the invariants                                |
| [surfaces.md](./surfaces.md)         | Building the API, CLI, live channel, and the board's batch on a flow's lane                                           |
| [carry-over.md](./carry-over.md)     | Writing the migration, the mock forge's new faces, and the proof                                                      |
