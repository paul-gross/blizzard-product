# Ingest contract

## The request

`POST /api/chunks` (`ChunkIngestRequest`) gains one optional field:

```python
class ChunkIngestRequest(BaseModel):
    tokens: list[str]
    project: str | None = None   # a project id or slug in the caller's tenant
```

The CLI's ingest verb, `blizzard hub chunk ingest`, gains `--project PROJECT`, an id or a slug. The board sends the
shell lens as `project` whenever the lens is on one ([surfaces.md](./surfaces.md)).

## Resolution, in order

The whole request resolves before anything is written, and rejects as a whole, as today.

1. **The project, if named.** `project` resolves to a project of the caller's tenant by [surfaces.md](./surfaces.md)
   §Resolving a project — `422` otherwise.
2. **Each token to a pointer.** Every token names its source — `{source}:{ref}`, `{source}#{ref}`, or the item's own URL
   — and resolves through the registry exactly as `WorkSourceRegistry.resolve` does today, within the tenant. Config
   already refuses two sources that could claim one token, so exactly one source ever does. A reference with no source
   (`212`, `BLZ-412`) is not accepted, as today.
3. **The project, if not named.** Every pointer's source is looked up among the tenant's links. When all of them are
   linked to exactly one and the same project, that project is used. Otherwise — a source linked to several projects, or
   sources whose single projects differ — the request is `422`, naming the candidate projects.
4. **Every source linked.** Each pointer's source must be linked to the resolved project (`422` naming the unlinked
   source). Linking is how a project declares where it draws from; ingest never links implicitly.
5. **One live holder.** `IngestService.ingest` (`hub/domain/chunk/ingest.py`) runs `require_unheld` over every pointer
   under `locked_work_refs`, unchanged (`IngestConflict`, answered `409` with the `ChunkIngestConflict` body). A work
   ref is held by at most one live chunk, so an item sits in at most one project at a time; once its holder is terminal
   it may be ingested again, into any project.

The chunk is minted by `mint_chunk` with the resolved `project_id`, and `chunks.project_id` is never written again.

## Pointers name a source

`chunk_work_refs`, `work_items`, `work_item_closures`, and every other table that stores a work ref keep the
`{source, ref}` shape, keyed by the source's name — which `epic:live-config` makes immutable
([records.md](../../../../delivered/live-config/spec/records.md)) — so a pointer never comes to mean another source.
Projects add nothing to a pointer: the project is the chunk's.

## Grouping

`GroupService.group` (`hub/domain/operations/queue.py`) refuses to fold a chunk into a survivor of a different project:
a new `ChunksInDifferentProjects` error, mapped to `409` beside `ChunkNotGroupable`. A fold within one project is
unchanged.

## Closing and annotating

Closing, annotating, and editing a work item go through its pointer's source binding — `WorkSourceRegistry.closer`,
`.annotator`, `.editor` — exactly as today. The chunk's project plays no part: a Jira item ingested into winter is
closed through the tenant's Jira source like any other. A chunk view's artifact branch links resolve through the
repositories its commits resolved to ([delivery.md](./delivery.md)), not through its first work ref's source.

## The built-in source

The built-in `hub` source is one tenant source, and every project is linked to it without a stored link. The source has
no `work_sources` row to point a link at — it is seated in the hub's code and its name is reserved — so the hub treats
it as linked to every project: resolution counts it among each project's sources, a project's source links list it as
linked, and unlinking it is refused. Every path that mints a hub work item and ingests it in one act names the project
explicitly, never relying on resolution step 3:

- `RunService` (`hub/domain/garden/runs/run.py`) ingests a run's item into the routine's own project;
- accepting a garden proposal ingests the minted item into the proposal's project;
- an agent's work-item proposal, materialized at delivery (`hub/domain/work_items/materialization.py`, through
  `materialize_create`), mints its item into the proposing chunk's project;
- an operator creating a hub item names its project: `blizzard hub item create` takes a required `--project`. The board
  offers no way to create a hub item.

Each item the built-in source mints records that project as its own (`work_items.project_id`), so a hub item belongs to
one project from the moment it exists, and the source stays one list for the whole tenant rather than a bucket per
project. `hub:<n>` refs stay unique per tenant (`work_item_sequence` is keyed per source, under the tenant key), so a
hub item's ref never collides across projects.
