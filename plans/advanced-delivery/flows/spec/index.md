# Flows specifications

The technical contracts behind `epic:advanced-delivery/flows`. Product intent begins at the [epic plan](../index.md);
implementation enters at the concern below. What a hub node's action does, how its outcome routes, and pending and its
poll are owned by the [actions epic](../../actions/spec/index.md) and `domain/artifacts/delivery.md`; landing only where
a chunk's project links is owned by `initiative:projects`'
[delivery contract](../../../../delivered/projects/hub/spec/delivery.md). This spec changes when a hub step runs and
what it waits behind; it changes nothing an action does once it runs.

This spec assumes the [actions](../../actions/spec/index.md) and [forges](../../forges/spec/index.md) epics have landed:
the four landings of the one and the five transition steps of the other, on `master`. Nothing here needs the cleanups
that follow either epic and remove what it deprecated, so flows starts whether or not they have followed. From actions:
a hub step is one of the hub's actions, taken a bounded pass at a time on the `ActionLoop`'s dedicated thread
(`hub/delivery/action_loop.py`), handing every effect it makes to an `IEffectSink` rather than writing it, and declaring
in its catalogue entry, `ActionSpec.lands` (`hub/domain/graph/actions.py`), whether it holds the branches its chunk
lands on and is handed forge handles; `RESERVED_WAITS` names the answers that are waits rather than choices, `pending`
alone until the `policies` epic. From forges: `ForgeRegistry` hands out each repository's forge roles, bundled as a
`RepositoryForge`, and `ACTION_REQUIREMENTS` (`hub/domain/actions/requirements.py`) answers the capabilities each action
requires of each repository, checked against `ForgeRegistry.offered(forge, repository=None)`. Forges checks those
capabilities and bundles the handles in `HubNodeExecutor` before a pass; this epic moves both into the flow driver's
visit build ([turns.md](./turns.md#building-the-visit)), since nothing calls the executor's `run` once the driver takes
turns. It builds on what is already true of the code: a repository carries an id (`repo_<ulid>`), a slug it may change,
and a name, and every reference holds the id, so a resource names its repository by id and every surface joins the
current slug and name when it reads; and each of a chunk's commit pointers resolves to its own repository record, with
its own base branch, forge, owner, and credential.

Paths are relative to `blizzard/src/blizzard/`.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`:

- `architecture/clean-architecture.md`'s `bzh:domain-package-layers` table, and `tests/test_layering.py`'s
  `_DOMAIN_PACKAGE_LAYERS` with it, gain a `delivery` row for `hub/domain/delivery/` at L4, importing `actions`,
  `chunk`, `graph`, `config`, and `artifact`. It is a domain package, distinct from the `hub/delivery/` edge beside the
  domain, which the layer walker tells apart by comparing whole path segments.
- `domain/artifacts/delivery.md` replaces its fleet-wide serialization paragraph with resources and flows, and corrects
  the scope the code already has: one slot per tenant, never one per fleet. Its "only a live, promoted chunk is driven"
  rule changes rather than stands: a chunk is driven at a hub node only while it is `delivering`, so a paused chunk,
  which hub-advance drives today, now waits for its resume ([levers.md](./levers.md)). That is the condition the
  driver's reconcile submits on ([verdict.md](./verdict.md#submitting)).
- `domain/work/statuses.md` says a `delivering` chunk is submitted to its flows or taking its turn in them.
- `domain/operations.md` gains the event kinds in [record.md](./record.md) and the `hub-action-stuck` escalation cause
  ([turns.md](./turns.md#a-turn-that-cannot-end)).
- `standards/hub-nodes/outcome-protocol.md` stops saying a `pending` answer releases the tenant's `hub_exec_slot`: it
  ends the turn ([turns.md](./turns.md#pending)).
- `architecture/crash-correctness/hub.md` replaces §The boot release of live hub-execution slots with the boot recovery
  of open turns ([carry-over.md](./carry-over.md#at-boot)) and registers the `flow.` crash-point family
  ([turns.md](./turns.md#crash-points)).
- `architecture/lanes.md`'s `bzh:lane-contract` names `FlowDriver` as the one hub lane that is not a `Sweep`: clause 5's
  "on the hub, every lane is a `Sweep`" gains the exception, and clause 4 says the driver catches each tick and each
  turn's task on its own ([turns.md](./turns.md#the-flow-driver)). `bzh:probe-gated-pass`'s table gains the driver's
  reconcile as a converging pass and its admission as work that comes due.
- `architecture/system-shape/exclusive-writes.md`'s single-process known debt replaces the boot slot release with the
  boot turn recovery and the driver's in-memory sweep of open turns ([turns.md](./turns.md#a-turn-that-cannot-end)).
- `architecture/system-shape/tenancy.md`'s list of hub-wide jobs that loop over open tenants replaces "boot slot
  release" with "boot turn recovery".
- `architecture/system-shape/store-facts.md` stops citing `hub_exec_slot.released_at` as its example of a released-at
  column, and `verification/blizzard/gaps.md` drops its entry on the slot's per-tenant lock.

`verification/blizzard/e2e-scenarios/board.md` gains the flow lanes. In `blizzard`'s own docs,
`docs/deployment/recovery.md` names the `flow.` crash points in place of `hubnode.after-poll.before-slot-release`, which
the actions epic left there until the slot retired, and `docs/upgrade.md` and `changes/unreleased/` gain the notes that
tell an operator what the second chunk changes for them and that the slot's table is deprecated
([carry-over.md](./carry-over.md#what-an-operator-meets)). Runners are untouched: the runner still calls hub-advance
while its chunk is `delivering` and reads nothing new from the answer.

The slot is described in docstrings the code keeps after it is gone, and each is rewritten with the change:
`hub/api/fleet.py`'s hub-advance route, `HubAdvanceResponse` (`wire/chunk.py`), `hub/domain/operations/stop.py`,
`hub/domain/operations/complete.py`, `hub/cli/chunk.py`, `hub/api/chunks.py`, `hub/domain/chunk/ports/lifecycle.py`, and
`hub/domain/garden/delivery/service.py`.

## Landing order

Every installation, the one blizzard is built with included, runs a pinned build and moves onto another only when its
operator moves it, but a release can be cut from `master` between any two of the chunks below, and a migration is frozen
the moment it merges. So the epic lands as four chunks, in order, and each leaves `master` releasable: deployable,
landing work as it did before, and passing `blizzard:compat` against every supported release. No chunk depends on one
after it, and each is proven before the next is minted. An installation may take the chunks one release at a time or
several in one, and is carried over either way ([carry-over.md](./carry-over.md)).

1. **The record, in the shadow of the slot.** The first migration ([carry-over.md](./carry-over.md#the-migrations))
   creates the six tables; `hub/domain/delivery/` with `FlowLine`, and with `PollPolicy`, which moves there from
   `hub/delivery/hub_node.py` because `FlowLine` reads it ([turns.md](./turns.md#who-is-next)); the store behind
   `IReadFlows` and `IWriteFlows`; `FlowDriver` running its reconcile alone; and the three flow invariants, beside
   `OneLiveExecSlot` ([record.md](./record.md#invariants)), arrive with it. The reconcile submits and supersedes exactly
   as [verdict.md](./verdict.md#the-reconcile) says, and the driver takes no turn: the slot still runs every pass, so a
   landing the slot makes closes its submission `superseded` on the next tick rather than with a verdict. While the slot
   runs, the reconcile writes nothing to a chunk itself: a visit its submit would refuse, and a chunk standing at a hub
   node that names no action, are left to the slot's run, which refuses or escalates them as the actions epic has it
   ([record.md](./record.md#what-the-record-says)). The line's shape is real, and `hub:flow-line-standing` holds it to
   live traffic before anything leans on it. Boot runs `release_orphaned_slots` and then `recover_turns` whole, and the
   first boot of this chunk is the one that admits chunks already mid-delivery in the order they reached their nodes.
2. **Turns replace the slot.** The driver takes turns, writes each pass's effects, routes answers, and ends turns as
   [turns.md](./turns.md) says; `ApplyService` and hub-advance wake it instead of running a step; stop, restart, and
   complete withdraw instead of releasing; `recover_turns` replaces `release_orphaned_slots`. Nothing acquires the slot
   from here, so its table is marked deprecated in the same change, the first half of removing it
   ([carry-over.md](./carry-over.md#the-deprecation-then-the-cleanup)). So that no release carries a hub step without
   its span, the trace corpus assembles each `hub exec` interval from turns here, still reading the slot's past rows
   while the table stands ([record.md](./record.md#invariants)); and since nothing can reach
   `hubnode.after-poll.before-slot-release` any longer, it leaves the crash-point registry and the kill-9 sweep as the
   `flow.` family joins them ([turns.md](./turns.md#crash-points)). `test.block` joins the test-fenced catalogue here,
   with the proofs that need it. The chunk carries the epic's change notes and its `docs/upgrade.md` section
   ([carry-over.md](./carry-over.md#what-an-operator-meets)).
3. **The slot's code retired.** `OneLiveExecSlot`, the slot's acquire, release, and count methods and their port,
   `DEFAULT_SLOT_STALE_AFTER`, and `IdPrefix.HUB_EXEC_SLOT` go, and the tests written against the slot are rewritten or
   removed ([carry-over.md](./carry-over.md#tests)). The table stays, deprecated and unwritten, and the trace corpus
   keeps reading its past rows.
4. **The surfaces.** The API, CLI, live channel, and the board's Delivery band ([surfaces.md](./surfaces.md)), and the
   documentation that names flows. Every read here is over rows the earlier chunks already write.

Chunks 1 through 4 are the epic, none of them is breaking, and the epic is delivered when they have landed. The slot's
table outlives them, deprecated: dropping it is a follow-up cleanup that belongs to no epic, a breaking change an
operator lets in only once a release has carried the deprecation
([carry-over.md](./carry-over.md#the-deprecation-then-the-cleanup)).

The `blizzard-context` changes listed above travel with the chunk whose code they describe. With the first go the layer
row, the lane-contract exception, `bzh:probe-gated-pass`'s row for the reconcile, and `domain/operations.md`'s
`turn-interrupted`, which the first chunk's boot recovery can already write ([carry-over.md](./carry-over.md#at-boot)).
With the second go the crash-correctness, exclusive-writes, and tenancy changes, `bzh:probe-gated-pass`'s row for
admission, `domain/artifacts/delivery.md`, `domain/work/statuses.md`, `standards/hub-nodes/outcome-protocol.md`, and
`domain/operations.md`'s `flow-landed` and `hub-action-stuck`, with `docs/deployment/recovery.md` in `blizzard`. The
gaps change goes with the third, the board scenario with the fourth, and the store-facts change with the cleanup that
drops the table, since the column it cites stands until then.

| Where                            | Read when                                                                                                                                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [turns.md](./turns.md)           | Replacing the slot — what a step holds, a turn, who is next on a branch, the hub's flow driver, pending, a turn that cannot end, crash points                                                           |
| [record.md](./record.md)         | Shaping the store — flows, submissions, turns, the append-only flow record, the landed event, invariants                                                                                                |
| [verdict.md](./verdict.md)       | Submitting and answering — the driver's reconcile, how a turn's outcome becomes its verdict, escalation, landing across several repositories                                                            |
| [levers.md](./levers.md)         | Pause, stop, restart, complete, requeue, and detach against a chunk that is submitted or mid-turn                                                                                                       |
| [surfaces.md](./surfaces.md)     | Building the API, CLI, live channel, and the board's flow lanes and card positions                                                                                                                      |
| [carry-over.md](./carry-over.md) | Writing the migrations — admitting chunks already mid-delivery at the head of their flows, deprecating the slot's table and the cleanup that drops it, what an operator meets, the tests, and the proof |
