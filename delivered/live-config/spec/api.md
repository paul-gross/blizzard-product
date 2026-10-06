# The configuration convention

One way every configured surface is changed, through every door: the API, the CLI, the board, and a declarative
document. The records it governs are [records.md](./records.md)'s and [secrets.md](./secrets.md)'s; this file owns the
verbs, what a patch means, how a document is read, and what an apply promises.

## Surfaces under the convention

| Surface        | Kind                                                |
| -------------- | --------------------------------------------------- |
| Work sources   | configured record                                   |
| Repositories   | configured record                                   |
| Secrets        | configured record, value write-only                 |
| Scopes         | configured record                                   |
| Routines       | configured record                                   |
| Graphs         | immutable mints with mutable flags                  |
| Work-item edit | not configuration — governed for patch meaning only |

## Verbs

| Verb   | HTTP                               | CLI                                     |
| ------ | ---------------------------------- | --------------------------------------- |
| create | `POST /api/<kind>`                 | `blizzard hub <noun> create`            |
| list   | `GET /api/<kind>?include_retired=` | `blizzard hub <noun> list`              |
| show   | `GET /api/<kind>/{key}`            | `blizzard hub <noun> show KEY`          |
| edit   | `PATCH /api/<kind>/{key}`          | `blizzard hub <noun> edit KEY --field…` |
| retire | `POST /api/<kind>/{key}/retire`    | `blizzard hub <noun> retire KEY`        |
| enable | `POST /api/<kind>/{key}/enable`    | `blizzard hub <noun> enable KEY`        |

There is no `DELETE` on a configured record; retirement stands where a deletion would. Every response is the record's
view, carrying its `revision`. Writes require `CONFIG_EDIT`, a new permission in `auth_core` granted to `admin` and
`superuser` beside `GRAPH_EDIT`; reads require `FLEET_VIEW`. The new kinds are `work-sources` (whose existing
`GET /api/work-sources` view gains the record fields additively), `repositories`, and `secrets`. Every CLI verb takes
`--json`, and sends `X-Blizzard-Door: cli`; the board sends `board`.

Status codes are fixed across kinds: 404 for an unknown key, 422 naming the offending field for a validation failure or
a reference to a missing or retired record, 409 for a revision mismatch.

## What a patch means

The hub has three meanings of `PATCH` today:

- `PATCH /api/routines/{routine_id}` (`RoutineEditRequest`) requires every field and a restated `name`, so it is a full
  replace under a patch verb.
- `PATCH /api/scopes/{slug}` (`ScopeEditRequest`) replaces its one field.
- `PATCH /api/work-sources/{source}/items/{ref}` (`WorkItemPatchRequest`) applies only the fields present, telling an
  omitted field from an explicit `null` through `model_fields_set` and `hub/domain/edit.py::UNSET`.

The convention is the third: **sparse merge**. An absent field is unchanged, a present field is set, and an explicit
`null` clears a nullable field and is refused on any other. Sparse merge is the only one of the three under which two
writers editing different fields do not overwrite each other, the only one an apply's per-field difference and
`epic:terraform-provider`'s per-attribute updates map onto without restating a record, and it already has an
implementation in the hub to reuse. Request models are `extra="forbid"`, every field optional.

A writer may send `If-Match: <revision>`; a mismatch is refused with 409 naming the current revision. An apply always
sends it.

## Documents

Each record kind has one wire model in `src/blizzard/wire/`, and its JSON Schema is served at
`GET /api/config/schema/{kind}`. A document reaches that model through the codec seam:

- **`IConfigCodec`** decodes bytes to a plain mapping and encodes a mapping to bytes, and names the media types and file
  extensions it serves (`bzh:pluggable-seams`). Validation happens after decoding, against the one wire model, so every
  format reaches the same validator and the same error messages.
- **The YAML binding** loads strictly: booleans are `true` and `false` only, scalars that YAML 1.1 would read as dates,
  sexagesimal numbers, or `yes`/`no` stay strings or are refused, and a duplicate key is an error. JSON is decoded by
  the same binding, being a subset of YAML, and also has a binding of its own for encoding.
- **Other formats** are later bindings behind the same seam.

The codec is chosen by `Content-Type` on the API and by file extension in the CLI. Graph definitions are read through it
too: `blizzard hub graph mint` and `GraphReconciliation` (`hub/graph_sync.py`) stop calling PyYAML directly.

## Apply

A declarative document states the records a hub should hold:

```yaml
version: 1
secrets: [gh-paul-gross]
work_sources:
  - name: blizzard
    provider: github
    locator: paul-gross/blizzard
    annotate: true
    secret: gh-paul-gross
repositories:
  - name: blizzard
    forge_api_url: https://api.github.com
    owner: paul-gross
    repo: blizzard
    base_branch: master
    secret: gh-paul-gross
```

`POST /api/config/apply?dry_run=<bool>` takes the document in any codec. `blizzard hub config apply FILE [--dry-run]`
sends it. The reconcile follows `graph sync`'s rule that nothing is written unless it changed:

| In the document | In the store       | Result                              |
| --------------- | ------------------ | ----------------------------------- |
| named           | absent             | create                              |
| named           | present, differing | sparse edit of the differing fields |
| named           | present, equal     | untouched — no revision moves       |
| named           | retired            | enable, then edit what differs      |
| not named       | any                | untouched                           |

An apply never retires: a document states records that should exist, not the whole of a hub, so one that names some of a
hub's records leaves the rest as they are, and retirement is always the explicit `retire` verb. A document is a door,
not an owner — a record an apply wrote takes edits through every other door, and the next apply of a document that still
states the old value restores it, which its dry run shows first.

`secrets` lists names that must exist and be active; values never appear in a document. The apply is one transaction:
any refusal — a validation failure, a missing secret — writes nothing. Its outcome is a list of `{kind, key, op, diff}`,
identical in shape under `dry_run`, which writes nothing. The rows it writes share one `apply_id` and carry
`door = apply`.

`blizzard hub config export [--format yaml|json]` encodes the hub's current records as a document, and
`blizzard hub config changes` pages the change log.

## Existing surfaces brought into line

Each lands within this epic; none of these routes is under `/api/fleet/...`, so the runner-reached wire is untouched
(`bzh:fleet-wire-additive`), and the board's client is regenerated from the changed schemas (`bzh:generated-client`).
The CLI on an operator's laptop is not upgraded when the hosted hub redeploys, so no change here breaks the requests an
older CLI sends. Sparse merge accepts the full, restating payloads today's routine and scope edits send — every field
present is a field set — so those stay compatible with no alias.

- **Routines.** `RoutineEditRequest` becomes sparse with every field optional. The restated `name` goes; a present
  `name` that differs is refused, keeping `RoutineNameImmutableError`. Routines gain `revision` and `config_changes`
  rows, and become declarable in a document.
- **Scopes.** `ScopeEditRequest` becomes the sparse model, with `description` optional. Scopes gain `revision` and
  `config_changes` rows, and become declarable in a document.
- **Graphs.** A graph is an immutable mint, so create is `mint` and an edit is a new mint; there is no patch of a
  definition. Its mutable flags move to the convention: `PATCH /api/graphs/{graph_id}` with `{follow_latest}` is the
  route, beside the existing retire and enable. `POST /api/graphs/{graph_id}/follow-latest` stays as an alias onto it,
  with its request and response unchanged, for a compatibility window of two minor releases after the PATCH ships, and
  is removed in a release whose changelog names it. Definitions decode through the codec seam.
- **Work items.** Their patch is already the convention's and is the reference implementation. A work item is work, not
  configuration, so it takes no revision, change log, or apply, and its `DELETE` stays.

## The rule in blizzard-context

The convention becomes a rule in a new spoke, `blizzard-context:architecture/system-shape/configuration.md`, routed from
`architecture/system-shape.md`, landing with this epic:

- **`bzh:configured-record`** — every configured record carries the verb set above, a sparse-merge patch, a revision and
  a `config_changes` row per write, and retirement in place of deletion. Detect: a `PATCH` request model with a required
  field; a `DELETE` route on a configured record; a configured write with no change row.
- **`bzh:config-codec`** — every document a hub ingests decodes through `IConfigCodec` and validates against its kind's
  one wire model. Detect: `yaml.safe_load` or `json.loads` of an ingested document outside a codec binding.
- **`bzh:secret-write-only`** — no route, response, log, span, event, or change row carries a secret value. Detect: a
  `SecretValue` reaching a controller, a serializer, or a logger.
- **`bzh:config-read-on-use`** — owned by [runtime.md](./runtime.md).
