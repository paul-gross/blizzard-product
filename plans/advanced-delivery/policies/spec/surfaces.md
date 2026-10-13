# Surfaces

The policy, its park timeout, and the landing order are repository fields, so they travel every door a repository's
other fields do, under the verbs `epic:live-config`'s [API contract](../../../../delivered/live-config/spec/api.md)
fixes. Nothing here adds a route.

## API

`wire/repository.py`'s write and read models gain `landing_policy`, `park_timeout_hours`, and `lands_after`. On a write,
`lands_after` takes ids or slugs and stores ids; on a read, it carries each as `{repository_id, slug, name}`, as a
record carries its forge. A repository created without a policy takes `merge-on-checks`. The refusals are the record's
own ([record.md](./record.md)): `422 forge_capability_missing`, `422 landing_order_unknown`, `422 landing_order_cycle`,
and `409` on retiring a repository another's order names. `GET /api/config/schema/repositories` lists the policies, and
for each the capabilities it requires, so a client can say why a choice is unavailable before the write refuses it. The
forge's document gains `offered`, its capabilities, beside its budgets ([budget.md](./budget.md#what-an-operator-sees)).

## CLI

`blizzard hub repo create` and `edit` gain `--policy fast-forward|merge-on-checks|park`, `--park-timeout <hours>`, and
`--lands-after <repository>`, repeatable, each an id or a slug; `edit`'s `--clear` accepts `park-timeout` and
`lands-after`. `blizzard hub repo show` prints the policy and the order. `blizzard hub forge show` prints the forge's
capabilities and its budgets' last readings. `contracts/cli/hub.json` is regenerated.

## The configuration document

A repository entry in `ConfigDocument` (`wire/config.py`) carries `landing_policy`, `park_timeout_hours`, and
`lands_after`, the last naming repositories by slug. Apply plans the order after every repository entry is planned, so a
document may declare a library and its consumer together, and resolves names through `PlannedRepositories` as other
references do; a cycle or an unknown name refuses the entry. An entry without `landing_policy` leaves a stored
repository's policy as it is and gives a new one `merge-on-checks`. Export writes all three, so a document exported
after this slice reproduces each repository's policy.

## Admin screens

The repository form (`web/projects/hub/src/app/admin/repositories/`) gains a policy picker, a park timeout shown only
for `park`, and a lands-after picker listing the tenant's other enabled repositories. A policy the chosen forge cannot
honour is shown disabled with the capabilities it lacks, read from the schema and the forge's `offered`. The repository
detail shows the policy and the order, each repository in it a link. The forge detail shows its capabilities and its
budgets. The client is regenerated from `openapi/hub.openapi.json`.

## The board and the flow lanes

A parked chunk's card shows its change and "Awaiting your merge" from the markers it already reads, and the chunk
detail's delivery rows show the same. The flows slice's place `state` (`FlowPlaceView`, `ChunkFlowPlaceView`) gains
`awaiting-merge`, so a parked chunk on a flow's lane reads "awaiting merge" with the change linked, and the lane's line
shows it as passed over rather than next. The card's one line under its node reads "awaiting merge on
`blizzard/master`". `blizzard hub flow show` prints the same state.
