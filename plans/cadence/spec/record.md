# The cadence record

## What a cadence holds

A cadence is a configured record of a project (`bzh:configured-record`), shaped as a scope is: a surrogate id, a slug
fixed at create, a display name, a `revision`, and a retired brake kept as append-only lifecycle facts. Its fields are
the rhythm and the chores that share it:

```yaml
slug: evening-architecture
name: Evening architecture sweep # the slug when absent
schedule:
  cron: "0 20 * * *" # a five-field cron expression: every day at 20:00
  zone: America/Chicago # an IANA zone; required with a schedule
spacing: 10d # the least time between an errand's runs
anchor: started # scheduled | started | finished; started when absent
chores:
  - key: architecture
    routine: architecture-sweep
    scopes: all # all | a list of slugs | absent for the routine's default scope
    mode: delta # full | delta; full when absent
    note: Weigh the hub's new sweep against the lanes rules.
    promote: true # true when absent
  - key: tests
    routine: test-gaps
    scopes: [hub-daemon, runner-daemon]
```

| Field              | Type                   | Default             | Meaning                                                                                                                                                                                                      |
| ------------------ | ---------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `slug`             | string                 | required            | The handle typed in URLs, the CLI, and config documents: lowercase letters, digits, and hyphens, as a scope's slug is (`ScopeSlug.parse`); unique in its project, retired cadences included; fixed at create |
| `name`             | string                 | the slug            | The display name the board shows; free text, editable, never a handle                                                                                                                                        |
| `schedule`         | table                  | absent              | The moments the cadence may fire ([due.md](./due.md) §The schedule)                                                                                                                                          |
| `schedule.cron`    | string                 | required in a table | A standard five-field cron expression — minute, hour, day of month, month, day of week — read as wall-clock time in `zone` ([due.md](./due.md) §The schedule)                                                |
| `schedule.zone`    | string                 | required in a table | The IANA zone the moments are read in; the hub holds no zone of its own, so a schedule never falls back on one                                                                                               |
| `spacing`          | string                 | absent              | A whole number and one unit, `m`, `h`, or `d` — `90m`, `36h`, `7d` — at least one minute ([due.md](./due.md) §The spacing)                                                                                   |
| `anchor`           | string                 | `started`           | Which moment of an errand's last run its spacing counts from ([due.md](./due.md) §The three anchors); meaningless, and refused, without one                                                                  |
| `chores`           | list of tables         | empty               | The chores the cadence keeps, in the order the operator wrote them                                                                                                                                           |
| `chores[].key`     | string                 | required            | The chore's handle within its cadence, a slug unique there                                                                                                                                                   |
| `chores[].routine` | string                 | required            | The routine the chore runs, by slug ([§Linked by slug](#linked-by-slug))                                                                                                                                     |
| `chores[].scopes`  | `all` or list of slugs | absent              | `all`: every scope the routine sweeps when the cadence fires; a list: those scopes; absent: one run over the routine's default scope                                                                         |
| `chores[].mode`    | string                 | `full`              | `full` or `delta`, settled per run exactly as a hand-started run's is                                                                                                                                        |
| `chores[].note`    | string                 | absent              | The prose appended to each run's charge as its *This run* section                                                                                                                                            |
| `chores[].promote` | boolean                | `true`              | Whether each run enters the ready queue at mint ([mint.md](./mint.md) §Promoting)                                                                                                                            |

A chore carries no urgency yet. `epic:urgency` adds it as one more chore field, and nothing here forecloses that: the
chore's table takes a nullable column, its document one optional key.

## Validation

Validation refuses only a cadence whose shape cannot mean anything, and never one whose references do not resolve. A
create, an edit, and an apply refuse, naming the field:

- a cadence with neither `schedule` nor `spacing`, since nothing would ever come due;
- an `anchor` stated with no `spacing`, or one other than `scheduled`, `started`, or `finished`;
- a `schedule` with no `cron`, a `cron` the cron library refuses to parse, one of six fields (a cadence is stated to the
  minute, never the second), one that names no moment in the next fifty years (`0 0 30 2 *`), or a `zone` missing or not
  in the zone database;
- a `spacing` that does not parse, or is under one minute;
- a chore missing `key` or `routine`, two chores sharing a key, a key or a listed scope that is not a valid slug
  (`ScopeSlug.parse`), an empty scope list, or a `mode` other than `full` or `delta`.

A routine no record carries, a retired one, a graph that no longer resolves, and a scope unknown, retired, or not linked
to the routine are all accepted, and shown as the attention the cadence needs ([surfaces.md](./surfaces.md) §Needs
attention). This is where a cadence parts from a routine's declaration, whose `RoutineDeclaration.check`
(`hub/domain/garden/declarations.py:66`) refuses an unknown graph or scope outright: a routine is the thing being
declared, while a cadence only points at things that may come and go beside it.

Two runtime dependencies arrive with the record. **`cronsim`** parses and evaluates the cron expression: it has no
dependencies of its own, iterates forward and backward from any instant, works with `zoneinfo`, and matches Debian's
cron exactly, daylight-saving handling included, so an operator's expectations from the cron they already know hold
here. It is reached only through one small schedule module of the cadence package, so the due rule never touches it and
a later change of library is a change to one module. **`tzdata`** carries the zone database, so `zoneinfo` resolves the
same names on a slim container image as on a workstation.

## Linked by slug

A chore names its routine and its scopes by slug, and the store keeps them that way rather than as ids. A routine's slug
is its lineage, unique in its project and never reused ([routine-slug.md](./routine-slug.md)), and a scope's slug is
unique in its project, so a slug is as steady a key as an id, and it lets a cadence be declared in the same document as
the routine it names, before either has an id. Resolution happens each time a cadence is read: through
`IReadRoutineRepository`'s bulk read by slug, `IReadScopeRepository.get` and `retired_slugs`, and
`IReadRoutineScopeRepository.list_scopes_for` — never cached across passes (`bzh:config-read-on-use`).

## A chore's identity

Each chore is minted a surrogate id, `chr_<ulid>` (a new `IdPrefix`), the first time its key appears in its cadence, and
the mark on every run it mints carries that id ([mint.md](./mint.md) §The errand's mark). An edit that keeps a chore's
key keeps its id, whatever else it changes — its routine, its scopes, its mode, its note, whether it promotes — so its
errands keep their last runs, as the plan decides. An edit that drops a key removes that chore, and a later edit that
writes the key again mints a new id, whose errands have never run. The key itself is never renamed: a chore renamed is
one chore removed and another added.

## The tables

Four tables, each tenant-keyed with a tenant-led index (`bzh:tenant-keyed-table`), shaped after `routines`,
`routine_scopes`, and `routine_lifecycle_facts` (`hub/store/schema.py:744`–`:797`):

- **`cadences`** — `cadence_id` (`cad_<ulid>`, hub-unique), `slug`, `name`, `schedule_cron` (nullable), `schedule_zone`
  (nullable), `spacing_minutes` (integer, nullable), `anchor` (nullable, read as `started`), `created_at`, `revision`,
  `tenant_id`, `project_id`; unique on `(project_id, slug)`.
- **`cadence_chores`** — `chore_id` (hub-unique), `cadence_id`, `key`, `routine_name`, `scope_selection` (`all`,
  `listed`, or `default`), `mode`, `note` (nullable), `promote`, `position`, `created_at`, `tenant_id`; unique on
  `(cadence_id, key)`.
- **`cadence_chore_scopes`** — `chore_id`, `scope_slug`, `tenant_id`; keyed on the pair, and holding rows only for a
  `listed` chore.
- **`cadence_lifecycle_facts`** — `id`, `cadence_id`, `retired`, `set_at`, `set_by`, `tenant_id`; newest wins, as a
  routine's do.

None of them holds when an errand is next due, what its last run did, or whether the cadence needs attention: each is
derived when read (`bzh:facts-not-status`). Removing a chore deletes its `cadence_chores` and `cadence_chore_scopes`
rows; the marks its runs carry name its `chore_id` without a foreign key, so a removed chore's runs stay readable as
runs of their routine. The tables arrive in one additive hub revision (`bzh:store-expand-contract`), with its colocated
migration test, and `contracts/store/hub.json` is regenerated; tenant teardown and the read census
(`tests/store_read_census.py`) learn them.

## The domain package

The cadence is a domain package of its own, `hub/domain/cadence/`, with its row in `_DOMAIN_PACKAGE_LAYERS`
(`tests/test_layering.py`) above `garden`, `work_items`, `chunk`, and `operations`, whose services it calls, and below
`config_apply`, which composes it (`bzh:domain-package-layers`). Its model holds the record, the chore, the schedule,
the spacing, the anchor, and the due rule as pure values and functions of an instant (`bzh:domain-orchestration-split`);
its services hold the verbs and the minting pass.

## The verbs

A cadence has the six verbs of every configured record — create, list, show, edit, retire, enable — each a write with
its `revision` and its `config_changes` row in one transaction, under a new `RecordKind.CADENCE`
(`hub/domain/config/changes.py:36`), the row naming the cadence's project. An edit is a sparse merge: a field left out
is untouched, and `chores`, when stated, is the whole list, matched to the stored chores by key. The change row's diff
names each chore added, removed, or changed by key.

| Route                                    | Verb    | Notes                                                      |
| ---------------------------------------- | ------- | ---------------------------------------------------------- |
| `POST /api/projects/{project}/cadences`  | create  | `201` with the cadence's view                              |
| `GET /api/projects/{project}/cadences`   | list    | the project's cadences, retired ones under `?retired=true` |
| `GET /api/cadences`                      | list    | the tenant's, or one project's under `?project=`           |
| `GET /api/cadences/{cadence_id}`         | show    |                                                            |
| `PATCH /api/cadences/{cadence_id}`       | edit    | `If-Match` on `revision`, `409` when stale                 |
| `POST /api/cadences/{cadence_id}/retire` | retire  | a brake: a retired cadence mints nothing                   |
| `POST /api/cadences/{cadence_id}/enable` | enable  |                                                            |
| `GET /api/cadences/{cadence_id}/errands` | errands | [surfaces.md](./surfaces.md) §Errands                      |

The routes sit in `hub/api/cadences.py`, mounted beside the routines' two routers (`hub/app.py:504`, `:541`), behind the
same guards. Every write needs `graph:edit`, as a routine's does. A write that leaves any chore of the cadence promoting
needs `chunk:control` as well, the permission promoting a chunk by hand already asks for: a cadence that admits work to
the queue is an admission, and only someone who may admit work may configure one.

## In the config document

The tenant's config document gains a `cadences` section, each entry a `CadenceEntry` — the cadence document above with
the `project` every project-owned entry names — on both `ConfigDocument` and `ConfigDocumentV2` (`wire/config.py`), its
schema served at `GET /api/config/schema/cadences`. `_entries` (`hub/api/config.py:258`) decodes it to a
`DeclaredCadence` (`hub/domain/config_apply/entries.py`), a `CadenceDeclaration` keyed `(project_id, slug)` implements
the `RecordDeclaration` protocol, and `reconcile` (`hub/domain/config/apply.py:285`) runs its section after routines, so
a document that declares a routine and the cadence that keeps it applies in one pass. `ConfigApplyStore._run`
(`hub/store/internal/config_apply_store.py`) gains the cadence's writes, and `GET /api/config/export` writes every
cadence not retired. Its `check` refuses only what [§Validation](#validation) refuses. As for every kind, an apply that
omits a cadence leaves it alone, and one that declares a retired cadence enables it (`bzh:config-apply`).
