# Surfaces

The policy and its park timeout are repository fields, so they travel every door a repository's other fields do, under
the verbs `epic:live-config`'s [API contract](../../../../delivered/live-config/spec/api.md) fixes. The landing order
belongs to a project's repository links, and gains doors of its own beside the links' own.

## API

`wire/repository.py`'s write and read models gain `landing_policy` and `park_timeout_hours`. A repository created
without a policy takes `merge-on-checks`. `RepositoryPatchRequest` refuses an explicit `null` on every field today; it
admits one on `park_timeout_hours` alone, which clears the timeout back to its default
([record.md](./record.md#the-fields)), and still refuses it on `landing_policy` and every other field. The refusal is
the record's own ([record.md](./record.md)): `422 forge_capability_missing`. `GET /api/config/schema/repositories` lists
the policies, and for each the capabilities it requires, so a client can say why a choice is unavailable before the
write refuses it. The forge's document gains `offered`, its capabilities, beside its budgets
([budget.md](./budget.md#what-an-operator-sees)).

A project's landing order is read and written beside its repository links (`hub/api/projects.py`), as the ready queue's
order is (`hub/api/queue.py`):

- `GET /api/projects/{project}/repository-links` carries each link's `position`, null when unranked, and lists the links
  in landing order.
- `PUT /api/projects/{project}/landing-order` takes the whole order as a list of repositories, each an id or a slug, and
  ranks them `1.0`, `2.0`, … in the list's order; a linked repository the list leaves out becomes unranked. It writes a
  fact only for a link whose position changes, so putting the order the project already has writes nothing. It stands
  apart from `repository-links/{repository}`, whose `PUT` already links one.
- `POST /api/projects/{project}/landing-order/position` moves one `repository` to just after `after_repository`, or to
  the front when that is null, or to unranked with `unranked: true`, writing one fact at the midpoint between its new
  neighbours, after renumbering the ranked ones when no midpoint is left
  ([record.md](./record.md#the-order-a-chunks-repositories-land-in)).

Both writes require `Permission.CONFIG_EDIT`, as the links do, and refuse a repository the project does not link
`422 repository_not_linked`. Each answers with the project's links in their new order.

## CLI

`blizzard hub repo create` and `edit` gain `--policy fast-forward|merge-on-checks|park` and `--park-timeout <hours>`;
`edit` gains `--clear park-timeout`, the first field `repo edit` can clear: it has no `--clear` today
(`hub/cli/repo.py`), so the flag arrives here, shaped as `source edit`'s, refusing a field given both a value and a
clear. An `edit --policy` that moves a repository off `park` clears its timeout with it
([record.md](./record.md#the-fields)). `blizzard hub repo show` prints the policy. `blizzard hub project show` prints
the project's repositories in landing order, each with its position or "unranked", and
`blizzard hub project order <project> <repository>…` sets the whole order, while
`blizzard hub project position <project> <repository> (--after <repository> | --first | --unranked)` moves one.
`blizzard hub forge show` prints the forge's capabilities and its budgets' last readings. `contracts/cli/hub.json` is
regenerated.

## The configuration document

A repository entry in `ConfigDocument` (`wire/config.py`) carries `landing_policy` and `park_timeout_hours`. An entry
without `landing_policy` leaves a stored repository's policy as it is and gives a new one `merge-on-checks`; an entry
that moves a repository off `park` clears its `park_timeout_hours`, and one that sets a timeout on a repository that
does not end up parked is refused, as the API refuses it.

A project entry (`ProjectDocument`) gains an optional `landing_order`, a list of repositories by id or slug, each one
the entry links. Apply plans it after the entry's links, so a document may link a repository and rank it together; it
ranks the listed repositories in the list's order and leaves every other linked repository unranked, writing a fact only
where a link's position changes, as the `PUT` does, so applying a document twice writes nothing the second time. A
repository it names that the project does not link refuses the entry `repository_not_linked`. An entry that omits
`landing_order` leaves the project's order as it stands. Export writes `landing_order` for every project with a ranked
repository, so a document exported after this epic rebuilds each project's order, and `epic:terraform-provider` has a
field to manage it through.

## Admin screens

The repository form (`web/projects/hub/src/app/admin/repositories/`) gains a policy picker and a park timeout shown only
for `park`. A policy the chosen forge cannot honour is shown disabled with the capabilities it lacks, read from the
schema and the forge's `offered`. The repository detail shows the policy. The forge detail shows its capabilities and
its budgets.

The project detail (`web/projects/hub/src/app/admin/projects/project-detail.ts`) shows the project's repositories, in
the link panel it already has (`project-link-panel.ts`), as one list in landing order that an operator drags to reorder,
the ranked ones first and an "Unranked" section after them, ordered by slug, into and out of which a repository may be
dragged. Each drop writes one position. The client is regenerated from `openapi/hub.openapi.json`.

## Escalations

The board's escalation sentences (`CAUSE_SENTENCES` in `web/projects/fleet/src/lib/chunk/chunk-detail/`), an exhaustive
record over `EscalationCause`, gain one each for `change-closed` ("A person closed the parked change without merging
it"), `branch-protected` ("The forge will not let the hub move this branch outside a pull request"), with the branch and
the forge's own message beside it from the escalation's detail, `merge-method-refused` ("This repository does not allow
merge commits, so the hub cannot merge its pull request"), and `park-timeout` ("The parked change has waited longer than
its repository allows").

## The board and the flow lanes

A parked chunk's card shows its change and "Awaiting your merge" from the markers it already reads, and the chunk
detail's delivery rows show the same. The flows epic's place `state` (`FlowPlaceView`, `ChunkFlowPlaceView`) gains
`awaiting-merge`, so a parked chunk on a flow's lane reads "awaiting merge" with the change linked, and the lane's line
shows it as passed over rather than next. The card's one line under its node reads "awaiting merge on
`blizzard/master`". `blizzard hub flow show` prints the same state.

The chunk detail's pending view says which wait a chunk is in. `PendingView` (`wire/chunk.py`) carries only the node's
name and the next poll's time today, so every hub-node wait reads alike; it gains the wait's kind, `pending` or
`awaiting-merge`, an open response vocabulary (`bzh:open-response-vocabulary`) so a later wait kind is additive, and the
moment the wait's run began, read from `hub_node_pending()` ([landing.md](./landing.md#awaiting-a-merge)). The detail
then reads "awaiting merge since …" for a parked chunk rather than the same line as a chunk whose checks are running.
