# Surfaces

A forge is configured through every door a repository is, with the verb set `epic:live-config`'s
[API contract](../../../../delivered/live-config/spec/api.md) fixes, and the records that name it change only in the
field that names it.

## API

`hub/api/forges.py`, under `/api`, with wire models in `wire/forge.py` (`ForgeSummary`, `ForgeDocument`,
`ForgePatchRequest`):

```text
GET   /forges
POST  /forges
GET   /forges/{forge}
PATCH /forges/{forge}
POST  /forges/{forge}/retire
POST  /forges/{forge}/enable
```

Writes need `Permission.CONFIG_EDIT` and an `If-Match` revision, and go through `ConfigAuthoring`
(`hub/domain/config/authoring.py`), which gains `create_forge`, `edit_forge`, `retire_forge`, and `enable_forge`. A
forge's document lists the repositories and work sources that name it. `GET /api/config/schema/{kind}` gains `forges`.
`{forge}` is an id or a slug, resolved as [record.md](./record.md) resolves one. Create takes `slug`, `name` (defaulting
to the slug), `kind`, `api_base`, and `web_base`; `PATCH` may change any of them but `kind`, and a slug it moves away
from becomes a former slug.

`wire/repository.py` gains `forge` beside `forge_api_url`, and `wire/work_source.py` gains `forge` beside `api_base` and
`web_base`: on a write, `forge` takes a forge's id or slug and stores its id; on a read, a record carries `forge_id`,
`forge_slug`, and `forge_name`, as a chunk carries its project's. A write naming an unknown or retired forge is refused
`422 forge_unavailable`; one naming a forge of the wrong kind for a work source's provider, `422 forge_kind_mismatch`.
Through the transition the old fields still work ([Old fields](#old-fields)).

## Old fields

The old address fields — `forge_api_url` on a repository, `api_base` and `web_base` on a work source — are accepted
through a transition and then retired ([carry-over.md](./carry-over.md)).

**Through the transition**, every door reads `forge` first and falls back to the old fields:

- A write that names `forge` uses it. A write that sends an old field as well is accepted only when the field agrees
  with the named forge, and refused `422 forge_conflict` when it does not.
- A write that sends only an old field resolves it to the tenant's forge with that API origin, and when none exists,
  creates one exactly as the migration slugs and names it, recording its creation in the change log under the same door
  and apply as the write that caused it. A work source's `web_base` that differs from its forge's is refused
  `422 forge_conflict`, since the web origin now belongs to the forge.
- Every read carries `forge` and the old fields derived from it, the latter marked deprecated in
  `openapi/hub.openapi.json`, so a client written before the slice keeps reading what it read.
- A write that leaned on an old field answers with a `Deprecation` header and a `warnings` entry naming the field and
  the forge it resolved to, and records a `config-old-field-used` event in `event_log`, so an operator can see when
  nothing still sends one.

**Once retired**, the old fields leave the wire models, reads carry only `forge`, and a write still sending one is
refused `422 field_moved_to_forge`, naming the field and the forge that now carries it, rather than silently dropped.

## CLI

`hub/cli/forge.py`, registered in `hub/cli/__init__.py`: `blizzard hub forge create|list|show|edit|retire|enable` —
`create <slug> [--name …] --kind … --api-base … [--web-base …]`, and `edit <forge> [--slug …] [--name …] …`, where
`<forge>` is an id or a slug. `blizzard hub repo` gains `--forge` beside `--forge-api-url`, and `blizzard hub source`
gains `--forge` beside `--api-base` and `--web-base`, each taking an id or a slug. Through the transition an old flag is
resolved as the API resolves its field and prints the API's warning; once retired, the old flags are removed.

## The configuration document

`ConfigDocument` (`wire/config.py`) and `hub/domain/config/apply.py` gain a `forges` section, each entry declared by its
slug, reconciled after the `secrets` check and before work sources and repositories, so a document may declare a forge
and the records that name it together. Apply never retires a forge, as it never retires anything. A repository entry
carries `forge`, an id or a slug, in place of `forge_api_url`, and a work-source entry carries `forge` in place of
`api_base` and `web_base`. Through the transition an entry carrying an old field is resolved as the API resolves it, a
forge it creates is reconciled before the entry, and the apply result lists the warning for each such entry; once
retired, the entry is refused `field_moved_to_forge`. Export writes the `forges` section first and only ever the new
fields, so a document exported during the transition is already one that survives its end.

## Admin screens

The admin shell (`web/projects/hub/src/app/admin/`) gains a Forges tab beside Repositories and Work sources, in
`admin/forges/` with the file set its siblings have — the page, the detail, `.query`, `.mutations`, and `.model` — built
from the shared `config-record-list`, `config-record-panel`, `config-form-dialog`, and `config-revisions`. A forge's
detail lists the records that name it, each a link. The repository and work-source forms replace their address fields
with a forge picker listing the tenant's enabled forges, filtered to the provider's kind for a work source. The client
in `web/projects/fleet/src/lib/api/hub/` is regenerated from `openapi/hub.openapi.json`.
