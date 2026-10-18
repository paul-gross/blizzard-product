# Surfaces

A forge is configured through every door a repository is, with the verb set `epic:live-config`'s
[API contract](../../../../delivered/live-config/spec/api.md) fixes, and the records that name it change only in the
field that names it.

## API

`hub/api/forges.py`, under `/api`, with wire models in `wire/forge.py` (`ForgeSummary`, `ForgeDocument`,
`ForgePatchRequest`, `ForgesListView`):

```text
GET   /forges
POST  /forges
GET   /forges/{forge}
PATCH /forges/{forge}
POST  /forges/{forge}/retire
POST  /forges/{forge}/enable
```

Reads need `Permission.FLEET_VIEW`, as repositories' and work sources' do. Writes need `Permission.CONFIG_EDIT` and an
`If-Match` revision, and go through `ConfigAuthoring` (`hub/domain/config/authoring.py`), which gains `create_forge`,
`edit_forge`, `retire_forge`, and `enable_forge`. A forge's document lists the repositories and work sources that name
it. `GET /api/config/schema/{kind}` gains `forges`. `{forge}` is an id or a slug, resolved by a `PathForge` dependency
in `hub/api/forge_params.py`, patterned on `repository_params.py` and `work_source_params.py`. Create takes `slug`,
`name` (defaulting to the slug), `kind`, `api_base`, and `web_base`; `PATCH` may change any of them but `kind`, and a
slug it moves away from becomes a former slug. A `PATCH` carrying a different `kind` is refused
`422 forge_kind_immutable`. Both addresses are stored canonical ([record.md](./record.md#the-table)), and an `api_base`
another forge of the tenant holds is refused `409 forge_origin_taken`, naming that forge.

`wire/repository.py` and `wire/work_source.py` gain one field on each side. A write carries `forge`, a forge's id or
slug, and stores its id; a read carries `forge_id`, `forge_slug`, and `forge_name`, as a chunk carries its project's.
Nothing reads a field called `forge`, and nothing writes the three it reads back. Through the transition
`RepositoryDocument.forge_api_url`, required today, becomes optional beside an optional `forge`, and a repository create
must carry exactly one of the two — neither is refused `422 forge_required`, and both are taken as the old-field rules
below say. A work source's create may carry neither, as it may leave `api_base` out today. A write naming an unknown or
retired forge is refused `422 forge_unavailable`; one naming a forge of the wrong kind for a work source's provider,
`422 forge_kind_mismatch`. A work source always names a forge ([record.md](./record.md#references)), so a create without
one is refused `422 forge_required` once the old fields retire, and `forge` cannot be cleared. Through the transition
the old fields still work ([Old fields](#old-fields)).

## Old fields

The old address fields — `forge_api_url` on a repository, `api_base` and `web_base` on a work source — are accepted
through a transition and then retired ([carry-over.md](./carry-over.md)).

**Through the transition**, every door reads `forge` first and falls back to the old fields:

- A write that names `forge` uses it. A write that sends an old field as well is accepted only when the field agrees
  with the named forge, and refused `422 forge_conflict` when it does not.
- A write that sends only an old field resolves it to the tenant's forge with that API origin, and when none exists,
  creates one exactly as the migration slugs and names it, suffixing its slug `-2` and onwards when another forge holds
  it, and recording its creation in the change log under the same door and apply as the write that caused it. Origins
  are compared canonical, so a trailing slash or a letter's case never creates a second forge, and two entries of one
  configuration document that spell one origin differently resolve to the one forge the apply plans for it. An address
  `canonical_origin` refuses is refused `422` on the field it came in, as any malformed field is.
- What a missing old field means follows what it means today, and so differs between creating a record and editing one.
  A work source *created* with neither `forge` nor `api_base`, as one on github.com may be, resolves to
  `https://api.github.com`, the factory's default today. On an *edit* — a `PATCH`, or a configuration document's entry
  for a record that already exists — an omitted field is left as stored, as `ConfigDocument` leaves every omitted field
  today, so an entry that names neither keeps the forge the source already names and never re-points an Enterprise
  source at github.com. An explicit `null` for `api_base` on an edit, or `blizzard hub source edit --clear api-base`,
  means what it means today, the github.com default: the source is re-pointed at the tenant's `api.github.com` forge,
  created if need be, and the write carries the warning below naming the forge it moved to, so the move is never silent.
  An empty or whitespace-only address is the same as an explicit `null`.
- An old field whose origin a retired forge holds is refused `422 forge_unavailable`, naming that forge: the origin is
  still claimed ([record.md](./record.md#retirement)), and an operator who wants the server back enables its forge
  rather than having a second one made behind them.
- Two writes that would create a forge for one origin at once meet at the forges table's `(tenant_id, api_base)`
  constraint. The forge's insert runs in its own savepoint (`conn.begin_nested()`), as the invitation store's racing
  insert does, since on Postgres a failed statement otherwise aborts the whole transaction; the one that loses rolls
  back to the savepoint, re-reads the forge the other created, and resolves to it, as a write that found it already
  there would have.
- A work source's `web_base` is canonicalized and compared against its forge's effective web origin — the forge's own
  `web_base`, or the origin derived from its `api_base` when that is null — and refused `422 forge_conflict` when it
  differs, since the web origin now belongs to the forge; a `web_base` sent as null, empty, or cleared means the
  forge's. This is the one write made before the epic that the transition refuses: a document or runbook that gives a
  work source a `web_base` of its own, different from every other record on that server, has nowhere to keep it once the
  web origin is the forge's. The refusal names the forge and its web origin, and the operator either sets that forge's
  `web_base` or drops the field ([carry-over.md](./carry-over.md#the-transition-then-retirement)).
- Every read carries `forge_id`, `forge_slug`, and `forge_name`, and the old fields derived from the forge, the latter
  marked deprecated in `openapi/hub.openapi.json`, so a client written before the epic keeps reading what it read. A
  repository's `forge_api_url` reads its forge's `api_base`. A work source's `api_base` reads its forge's `api_base`,
  never null — one stored null before the epic now reads `https://api.github.com`, the address it was always resolved to
  — and its `web_base` reads the forge's effective web origin.
- A write that leaned on an old field answers with a `Deprecation` header and a `warnings` entry naming the field and
  the forge it resolved to — the responses `RepositorySummary` and `WorkSourceSummary` (`wire/repository.py`,
  `wire/work_source.py`) and the configuration apply's result each gain a `warnings` list for it, empty on every other
  answer, while `RepositoryDocument` and `WorkSourceDocument` stay request bodies and the configuration document's
  schema and gain nothing — and records a `config-old-field-used` event in `event_log`, so an operator can see when
  nothing still sends one. The event is documented in `docs/deployment/observability.md` beside
  `forge-capability-missing`.

**Once retired**, the old fields leave the wire models, reads carry only `forge_id`, `forge_slug`, and `forge_name`, and
a write still sending one is refused `422 field_moved_to_forge`, naming the field and the forge that now carries it,
rather than silently dropped.

## CLI

`hub/cli/forge.py`, registered as `"forge"` in the lazy `_COMMANDS` map of `hub/cli/__init__.py`:
`blizzard hub forge create|list|show|edit|retire|enable` —
`create <slug> [--name …] --kind … --api-base …
[--web-base …]`, and `edit <forge> [--slug …] [--name …] …`, where
`<forge>` is an id or a slug. `blizzard hub repo` gains `--forge` beside `--forge-api-url`, whose `required=True` on
`repo create` (`hub/cli/repo.py`) gives way to requiring exactly one of the two, and `blizzard hub source` gains
`--forge` beside `--api-base` and `--web-base`, each taking an id or a slug; `source edit`'s `--clear` keeps `api-base`
and `web-base` through the transition and only `secret` once they retire, since every provider a work source can have
today is a forge, and a forge-backed source always names one. Through the transition an old flag is resolved as the API
resolves its field and prints the API's warning; once retired, the old flags are removed. `contracts/cli/hub.json` is
regenerated with each release.

## The configuration document

`ConfigDocument` (`wire/config.py`) and `hub/domain/config/apply.py` gain a `forges` section, each entry declared by its
slug as repository and work-source entries now are, reconciled after the `secrets` check and before work sources and
repositories, so a document may declare a forge and the records that name it together. It follows their shape: a
`ForgeDeclaration` whose `forge_id` is minted at the edge for a forge the document creates (`_Forges.of` beside
`_Repositories.of` and `_WorkSources.of` in `hub/api/config.py`), a `PlannedForges` lookup in
`hub/domain/config/forges.py` through which repository and work-source entries resolve a forge declared in the same
document — patterned on `PlannedRepositories` and `PlannedWorkSources` (`hub/domain/projects/declarations.py`), but
beside the forge record, since the config apply rather than the project plan reads it — and
`StoredConfig.former_forges`, so a declared slug another forge has left is refused by `_require_unclaimed` as a
repository's is. Apply never retires a forge, as it never retires anything, and refuses an entry whose `kind` differs
from the stored forge's, `forge_kind_immutable`, as the API does. A repository entry carries `forge`, an id or a slug,
in place of `forge_api_url`, and a work-source entry carries `forge` in place of `api_base` and `web_base`. Through the
transition an entry carrying an old field is resolved as the API resolves it, a forge it creates is reconciled before
the entry, and the apply result lists the warning for each such entry; once retired, the entry is refused
`field_moved_to_forge`. Export writes the `forges` section right after `projects`, which stays first, and only ever the
new fields, each entry's `forge` written as the forge's slug as every reference in the document is, so a document
exported during the transition is already one that survives its end.

## Admin screens

The admin shell (`web/projects/hub/src/app/admin/`) gains a Forges tab beside Repositories and Work sources, in
`admin/forges/` with the file set its siblings have — the page, the detail, `.query`, `.mutations`, and `.model` — built
from the shared `config-record-list`, `config-record-panel`, `config-form-dialog`, and `config-revisions`, and showing
each forge's slug and name as the Repositories and Work sources tabs now show theirs; the kind enumerations it joins are
listed in [record.md](./record.md#the-change-log). A forge's detail lists the records that name it, each a link. The
repository and work-source forms replace their address fields with a forge picker listing the tenant's enabled forges,
filtered to the provider's kind for a work source. The shared form dialog has no picker to offer: `ConfigFieldKind`
(`admin/config-edit.model.ts`) is `'text' | 'password' | 'checkbox'`, so it gains `'select'`, whose field carries its
options, and `config-form-dialog` renders it; the forge picker is its first use. The change log's revisions view shows a
`forge` field by its `new_label` and `old_label` ([record.md](./record.md#the-change-log)). The client in
`web/projects/fleet/src/lib/api/hub/` is regenerated from `openapi/hub.openapi.json`.
