# Policies specifications

The technical contracts behind `epic:advanced-delivery/policies`. Product intent begins at the [epic plan](../index.md);
implementation enters at the concern below. Configured records, their revisions, and the change log are owned by
`epic:live-config`'s [records](../../../../delivered/live-config/spec/records.md) contract; landing only where a chunk's
project links by `epic:projects/hub`' [delivery contract](../../../../delivered/projects/hub/spec/delivery.md); the
forge record, the landing seam, and the capability check before a pass by the [forges epic](../../forges/spec/index.md);
and when a visit takes its turn, and how its verdict reaches its chunk, by the [flows epic](../../flows/spec/index.md).
This spec decides how a repository's work lands once its turn comes; it changes nothing about when the turn comes.

This spec assumes the [actions](../../actions/spec/index.md) and [flows](../../flows/spec/index.md) epics have landed,
and the [forges](../../forges/spec/index.md) epic's transition release — the seam, the forge record, and `land` ported
onto them — not its later release retiring the old address fields, which nothing here needs: landing is the hub's `land`
action, taken a pass at a time; it speaks every forge through the roles `ForgeRegistry` hands out — `IForgeRefs`,
`IForgeChanges`, and `IForgeChecks`; and a visit lands in turns on the flow of each branch it holds. It builds on what
the code already does: each of a chunk's commit pointers resolves to its own repository record, and the `land` action
lands each against that record's own server, owner, base branch, and credential. A chunk spanning two GitHub servers
lands today; this epic adds a policy per repository and the order a project lands its repositories in. GitHub is the
only forge kind it lands on; the seam it grows is forge-neutral, and the second kind is the
[gitlab epic](../../gitlab/spec/index.md)'s.

Paths are relative to `blizzard/src/blizzard/`.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, `domain/artifacts/delivery.md` trades "the policy is
the script's" for the policy being the repository's, names the three policies and the project's order a chunk's
repositories land in, and says a parked change is noticed by polling; `standards/hub-nodes/outcome-protocol.md` gains
the reserved `awaiting-merge` beside `pending` and measures the poll timeout from the first pending attempt since the
last wait of another kind; `domain/work/statuses.md` says a `delivering` chunk may be awaiting a person's merge;
`architecture/system-shape/configuration.md` learns the repository's policy fields; `domain/projects.md` says a project
orders the repositories it links; `domain/operations.md` gains the event kinds and the escalation causes below;
`architecture/crash-correctness/hub.md` registers the policies' crash points ([landing.md](./landing.md#crash-points));
and `verification/blizzard/e2e-scenarios/delivery-policy.md` gains a scenario per policy. `docs/deployment/` gains a
page on choosing a policy, including the CI a fast-forwarding repository needs. `blizzard-mock`'s forge grows what the
proofs need ([seam.md](./seam.md#the-mock)).

## Landing order

The instance runs a pinned build, never `master` itself, but a release can be cut from `master` between any two steps,
and the fleet that lands this epic lands its own work through the very `land` it rewrites. So the epic arrives as six
steps, each a chunk or a few, in this order, and each leaves `master` releasable: landing as it did before the step, and
able to walk its store back to the step before it ([carry-over.md](./carry-over.md#walking-back)). No step depends on
one after it, and no two are meant to ship together.

1. **The mock's forge** ([seam.md](./seam.md#the-mock)), in `blizzard-mock` alone. Every addition is new — routes,
   parameters, headers, levers — or answers as GitHub does where the mock had answered loosely, so the hub's existing
   proofs pass against it unchanged — `blizzard`'s own and every supported release's — and every later step has the
   forge its proofs need ([seam.md](./seam.md#what-the-mock-must-keep)). It merges first because the hub's steps cannot
   be proven without it. Nothing in it needs the earlier epics' code, so it may land ahead of them, but not while the
   forges epic's own additions to the mock are being built, which change the same files.
2. **The seam and the cheaper pass** ([seam.md](./seam.md#the-seam-grows), [budget.md](./budget.md#the-cost-of-a-pass)).
   The new seam operations, their capabilities, and their GitHub implementations; conditional reads and the compare
   store; the already-merged lookup as one filtered read; and the shipped graphs polling every minute. `land` still
   merges once checks pass, now more cheaply, and nothing calls the new operations yet. No migration.
3. **The budget** ([budget.md](./budget.md#the-budget)): `ForgeBudgets`, priorities carried by the registry's role
   handles and the work-source family, the close drain's `deferred` outcome, and the two event kinds, with the budget's
   migration ([carry-over.md](./carry-over.md#the-migrations)). Nothing is turned away until a forge reports itself low,
   so a healthy fleet sees no difference.
4. **`land` as a runner of policies** ([landing.md](./landing.md#land-runs-policies), [record.md](./record.md)), with
   `merge-on-checks` its only policy. The policies' migration, the repository's policy fields, the landing order, the
   visit's terms, combining, the poll row's kind, and the repository and project doors — API, CLI, and configuration
   document — for the policy and the order. `LandingPolicy` has one member here, so the doors refuse any other as
   unknown, and every repository still lands by merging once checks pass; the one change a fleet sees is the order a
   chunk's repositories land in, slug order until its project sets one
   ([carry-over.md](./carry-over.md#the-migrations)).
5. **`fast-forward` and `park`** ([landing.md](./landing.md#the-three-policies)). `LandingPolicy` gains its two members
   and `POLICY_CAPABILITIES` their rows; the reserved `awaiting-merge`, the escalation causes, and their board sentences
   arrive with them. The columns step 4 added already hold any policy, so its migration changes no schema; it is the
   revision whose downgrade keeps an older hub from meeting a policy or a wait it cannot read
   ([carry-over.md](./carry-over.md#the-migrations)). Nothing changes for a repository until an operator sets one of
   them.
6. **The board** ([surfaces.md](./surfaces.md#admin-screens)): the repository form's policy picker, the project's
   drag-to-order list, the forge detail's capabilities and budgets, and the flow lane's `awaiting-merge`, each reading
   doors earlier steps opened.

The `blizzard-context` and `docs/` changes listed above land with the step whose behaviour they describe.

| Where                            | Read when                                                                                                              |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| [record.md](./record.md)         | Shaping the repository's policy fields, a project's landing order, the terms a visit takes, and the capability check   |
| [landing.md](./landing.md)       | Building `land` as a runner of policies — the three policies, combining, escalations, awaiting a merge, crash points   |
| [seam.md](./seam.md)             | Building the seam operations this epic adds, their GitHub implementations, and the mock forge's additions              |
| [budget.md](./budget.md)         | Making a landing pass cheap, sharing one rate budget per key, and slowing down before the forge refuses                |
| [surfaces.md](./surfaces.md)     | Building the API, CLI, configuration document, admin screens, escalation sentences, and the flow lane's awaiting state |
| [carry-over.md](./carry-over.md) | Writing the migrations, walking a store back between steps, judging each step's compatibility, and the proof           |
