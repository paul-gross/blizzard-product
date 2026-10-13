# Policies specifications

The technical contracts behind the policies slice of `epic:advanced-delivery`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. Configured records, their revisions, and the
change log are owned by `epic:live-config`'s [records](../../../../delivered/live-config/spec/records.md) contract;
landing only where a chunk's project links by `epic:projects`'
[delivery contract](../../../../delivered/projects/hub/spec/delivery.md); the forge record, the landing seam, and the
capability check before a pass by the [forges slice](../../forges/spec/index.md); and when a visit takes its turn, and
how its verdict reaches its chunk, by the [flows slice](../../flows/spec/index.md). This spec decides how a repository's
work lands once its turn comes; it changes nothing about when the turn comes.

This spec assumes the [actions](../../actions/spec/index.md), [forges](../../forges/spec/index.md), and
[flows](../../flows/spec/index.md) slices have landed: landing is the hub's `land` action, taken a pass at a time; it
speaks every forge through the roles `ForgeRegistry` hands out — `IForgeRefs`, `IForgeChanges`, and `IForgeChecks`; and
a visit lands in turns on the flow of each branch it holds. It builds on what the code already does: each of a chunk's
commit pointers resolves to its own repository record, and the land script lands each against that record's own server,
owner, base branch, and credential. A chunk spanning two GitHub servers lands today; this slice adds the second kind of
forge, and a policy per repository.

Paths are relative to `blizzard/src/blizzard/`.

The slice reaches beyond `blizzard`'s source. In `blizzard-context`, `domain/artifacts/delivery.md` trades "the policy
is the script's" for the policy being the repository's, names the three policies and the project's order a chunk's
repositories land in, and says a parked change is noticed by polling; `standards/hub-nodes/outcome-protocol.md` gains
the reserved `awaiting-merge` beside `pending`; `domain/work/statuses.md` says a `delivering` chunk may be awaiting a
person's merge; `architecture/system-shape/configuration.md` learns the repository's policy fields; `domain/projects.md`
says a project orders the repositories it links; `domain/operations.md` gains the event kinds and the escalation causes
below; and `verification/blizzard/e2e-scenarios/delivery-policy.md` gains a scenario per policy and one for GitLab.
`docs/deployment/` gains a page on choosing a policy, including the CI a fast-forwarding repository needs.
`blizzard-mock`'s forge grows what the proofs need ([gitlab.md](./gitlab.md#the-mock)).

| Where                            | Read when                                                                                                                |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| [record.md](./record.md)         | Shaping the repository's policy fields, a project's landing order, and the capability check when a policy is set         |
| [landing.md](./landing.md)       | Building `land` as a runner of policies — the three policies, combining, escalations, awaiting a merge, crash points     |
| [gitlab.md](./gitlab.md)         | Building the GitLab adapter, the seam operations this slice adds, and the mock forge's new faces                         |
| [budget.md](./budget.md)         | Making a landing pass cheap, sharing one rate budget per key, and slowing down before the forge refuses                  |
| [surfaces.md](./surfaces.md)     | Building the API, CLI, configuration document, admin screens, escalation sentences, and the flow lane's awaiting state   |
| [carry-over.md](./carry-over.md) | Writing the migration — every repository given the policy it lands by today, the positions, the poll kind, and the proof |
