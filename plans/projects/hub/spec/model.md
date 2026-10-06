# Model contract

## The project record

A project is a configured record inside a tenant. It follows `epic:live-config`'s convention for configured records
([records.md](../../../../delivered/live-config/spec/records.md)): a stable id, a revision that moves with every change,
the create, read, and edit verbs, and change facts that record who changed it and through which door.

```text
projects
  project_id    String  PK          proj_<ulid>
  tenant_id     String  NOT NULL    the tenant key (multi-tenancy store.md)
  slug          String  NOT NULL    the project's handle; editable
  name          String  NOT NULL    display name; editable
  description   Text    NOT NULL
  revision      Integer NOT NULL
  created_at    UtcDateTime NOT NULL
  UNIQUE (tenant_id, slug)          uq_projects_tenant_slug

project_former_slugs                  a slug a project moved away from
  tenant_id     String  NOT NULL
  slug          String  NOT NULL
  project_id    String  FK projects NOT NULL
  left_at       UtcDateTime NOT NULL
  PRIMARY KEY (tenant_id, slug)       at most one project holds a given former slug
```

A project cannot be retired: this slice gives it no retire or enable verb and no lifecycle facts. Retiring a project is
left for later ([slice plan](../index.md) §Left for later).

A project carries three things that people and machines tell it apart by, and each has one job:

- **`project_id`** is the project's identity: every `project_id` column, link, runner declaration, change fact, and
  internal reference uses it, and nothing stores a slug or a name in its place.
- **`slug`** is the handle people type and read in URLs, CLI flags, API paths, and runner configuration. It is 1–64
  characters of lowercase ASCII letters, digits, and hyphens, a hyphen only ever between two letters or digits — no
  leading or trailing hyphen, no double hyphen, no other symbol (`^[a-z0-9]+(-[a-z0-9]+)*$`). It is unique within the
  tenant, which `uq_projects_tenant_slug` enforces, and two tenants may each hold the same slug.
- **`name`** is the display name: free text, 1–128 characters with no leading or trailing whitespace, unique nowhere.
  Nothing resolves a name; it is only ever shown.

Changing a project's slug records the slug it leaves in `project_former_slugs`, replacing any earlier holder of that
former slug. A slug that is some project's former slug — another project's, or the project's own — may be taken by any
project of the tenant; taking it removes its `project_former_slugs` row in the same write, so from then on it resolves
to its new holder and a slug is never both current and former. Changing the slug or the name changes nothing that
references the project. Where a slug is accepted as input — a URL, a CLI flag, an API path, a runner declaration — it is
resolved to the id at the boundary ([surfaces.md](./surfaces.md) §Resolving a project).

## Every tenant starts with a project

A tenant holds a `default` project from the moment it exists, so it can ingest, hold a routine, and give a runner a
project to declare before anyone has configured anything. Creating a tenant creates it in the same transaction
(multi-tenancy [administration.md](../../../multi-tenancy/hub/spec/administration.md) §Creating a tenant): slug
`default`, display name `Default`, attributed to whoever created the tenant, and linked to the built-in `hub` source as
every project is, with no stored link. The carry-over gives the same project to every tenant that exists when this slice
lands ([carry-over.md](./carry-over.md)), so a fresh hub's empty `default` tenant holds one too. Nothing else sets it
apart: its slug and name change like any other project's.

## Links to work sources and repositories

Work sources and repositories are tenant records owned by live-config
([records.md](../../../../delivered/live-config/spec/records.md)); this slice adds no column to either. A project draws
from a source, and may land in a repository, through a link. A project draws from a source through a source link:

```text
project_source_links
  tenant_id     String  NOT NULL      the tenant key (multi-tenancy store.md)
  project_id    String  FK projects   NOT NULL
  source_name   String  NOT NULL      immutable (live-config records.md)
  revision      Integer NOT NULL
  PRIMARY KEY (project_id, source_name)
  FOREIGN KEY (tenant_id, source_name) REFERENCES work_sources (tenant_id, name)

project_source_link_facts             append-only; linked/unlinked derives from the newest fact
  id, project_id, source_name, linked Boolean, set_at, set_by
```

A link is a configured record of its own: it carries a revision and its own change facts, and unlinking is the link's
retirement. A link carries nothing beyond the fact that the project draws from the source: what the project may ingest
from it is never narrowed. The built-in `hub` source has no row here: it has no `work_sources` row for the foreign key
to name, and every project is linked to it without one ([ingest.md](./ingest.md) §The built-in source).

A project may land in a repository through a repository link:

```text
project_repository_links
  tenant_id        String  NOT NULL      the tenant key (multi-tenancy store.md)
  project_id       String  FK projects   NOT NULL
  repository_name  String  NOT NULL      immutable (live-config records.md)
  revision         Integer NOT NULL
  PRIMARY KEY (project_id, repository_name)
  FOREIGN KEY (tenant_id, repository_name) REFERENCES repositories (tenant_id, name)

project_repository_link_facts          append-only; linked/unlinked derives from the newest fact
  id, project_id, repository_name, linked Boolean, set_at, set_by
```

A repository link grants exactly one thing, that the project's chunks may land in the repository
([delivery.md](./delivery.md)). Like a source link it is a configured record with a revision and change facts, and
unlinking is its retirement. A repository linked by no project is legal and receives nothing. The link is where
`epic:advanced-delivery` may attach a project-level default over the repository's own landing policy, should one earn
its place.

## Which tables carry the project

Every table carries `tenant_id` under the multi-tenancy contract. `project_id` is added only where a record's project
cannot be reached through a chunk, or where a hot read filters on it:

| Table                  | `project_id`          | Why                                                                                           |
| ---------------------- | --------------------- | --------------------------------------------------------------------------------------------- |
| `chunks`               | NOT NULL              | Set once at mint ([ingest.md](./ingest.md)); every chunk-owned fact derives from it           |
| `scopes`               | NOT NULL              | Project-owned; `(project_id, slug)` unique                                                    |
| `routines`             | NOT NULL              | Project-owned; `(project_id, name)` unique                                                    |
| `findings`             | NOT NULL              | A routine finding has no chunk; a review finding's chunk agrees with it                       |
| `finding_sets`         | NOT NULL              | Read by `(project_id, routine_name, scope_slug)`, not via its chunk                           |
| `garden_proposals`     | NOT NULL              | An operator-origin proposal has no chunk                                                      |
| `work_item_runs`       | NOT NULL              | A run's routine name and scope label only mean something inside a project                     |
| `work_items`           | NOT NULL              | Every row is a built-in `hub` source item, minted into one project ([ingest.md](./ingest.md)) |
| `config_changes`       | NULL                  | The project a changed record belongs to; null for a tenant-wide record (below)                |
| `runner_registrations` | — (`projects` column) | The served declaration ([eligibility.md](./eligibility.md))                                   |

Everything else that hangs off a chunk — `chunk_work_refs`, `transitions`, `artifacts`, `lease_facts`, `usage_facts`,
`questions`, `decisions`, `escalations`, the delivery and closure facts, transcripts, and `event_log` rows that name a
chunk — reaches its project through `chunks.project_id` and gains no column. A read that filters such rows by project
joins through `chunks`, using the `(tenant_id, project_id, minted_at, chunk_id)` index this slice adds beside
`ix_chunks_minted_at_chunk_id`. A hot read that proves the join too costly earns a denormalized column in its own
change, never speculatively.

`config_changes`, the change log `epic:live-config` writes for every configured record, gains a nullable `project_id`:
set for a project, a project's links, and a scope or routine, whose `record_key` — a slug or a name — means something
only inside its project; null for a tenant-wide record — a work source, a repository, a secret. Admin → Changes filters
by it ([surfaces.md](./surfaces.md) §What each view does with the lens).

## Declaring projects in a configuration document

`epic:live-config`'s apply document (`hub/domain/config/apply.py`), which today declares `work_sources`, `repositories`,
`scopes`, and `routines`, gains a `projects` section. Each entry declares a project by slug — its name and description,
the sources it links, and the repositories it links — and reconciles before the scopes and routines, so a scope or
routine may name a project the same document declares. Each scope and routine entry gains a `project`, an id or a slug;
an entry without one belongs to the tenant's `default` project, so a document written before this slice applies
unchanged to a carried-over hub. A repository the document declares is linked to no project until a project entry links
it.

## Scope and routine identity

`scopes` already carries the surrogate `scope_id` the multi-tenancy
[store contract](../../../multi-tenancy/hub/spec/store.md) gives it, with every scope foreign key pointing at that id.
This slice only narrows uniqueness: `uq_scopes_tenant_slug (tenant_id, slug)` becomes `(project_id, slug)`, and
`uq_routines_tenant_name (tenant_id, name)` becomes `(project_id, name)`. A routine's default scope and every
`routine_scopes` row must name a scope of the routine's own project, enforced on write. Columns that carry a scope slug
or routine name as a denormalized label — `findings.scope_slug`, `findings.routine_name`, `finding_sets`,
`garden_proposals.routine_name`, `work_items.routine_name` — keep the label and resolve it within the row's own
`project_id`.

Mint-on-name holds within the project (`domain/routines-and-scopes.md` §Mint-on-name): naming a slug no scope of that
project holds mints it there, and the same slug in another project is a different scope. A review round's deferred
finding mints its scope in the reviewed chunk's project.

## Across projects

- **A garden proposal cites only its own project's findings.** Every write of a `garden_proposal_findings` row — garden
  delivery's materialization (`hub/domain/garden/delivery/materialize.py`) and the operator's proposal routes — checks
  that the finding's `project_id` is the proposal's, and refuses the write otherwise: a delivery that cites another
  project's finding fails its hub step naming the finding, and an operator's request answers `422` naming it.
- **A chunk may depend on a chunk of another project.** Dependencies (`chunk_dependencies`) stay tenant-wide and are
  untouched; a runner serving one project waits on another project's chunk as it waits on any blocker today.

## Invariants

The invariant checker (`bzh:invariant-checker`) gains:

- every chunk's `project_id` names a project of the chunk's tenant;
- every `routine_scopes` row and every routine's default scope join a routine and a scope of the same project, and every
  finding and finding set names a scope of its own `project_id`;
- a review-sourced finding's `project_id` equals its `raised_by_chunk_id` chunk's;
- every `garden_proposal_findings` row joins a proposal and a finding of the same project;
- every `work_items` row's `project_id` names a project of its tenant;
- every source a chunk's work refs name is linked to the chunk's project, or was when the chunk was minted — the link's
  facts answer which.
- every `delivery_repo_landed` row whose `repository_name` is set names a repository the chunk's project linked when the
  row was written — the link's facts answer which ([delivery.md](./delivery.md) §Landing).
