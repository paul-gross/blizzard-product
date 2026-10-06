# Ingest contract

## The request

`POST /api/chunks` (`ChunkIngestRequest`) gains one optional field:

```python
class ChunkIngestRequest(BaseModel):
    tokens: list[str]
    project: str | None = None   # a project id or name in the caller's tenant
```

The CLI's ingest verb gains `--project PROJECT`, an id or a name. The board sends the shell lens as `project` whenever
the lens is on one ([surfaces.md](./surfaces.md)).

## Resolution, in order

The whole request resolves before anything is written, and rejects as a whole, as today.

1. **The project, if named.** `project` resolves to a non-retired project of the caller's tenant by
   [surfaces.md](./surfaces.md) §Resolving a project — `422` otherwise.
2. **Each token to a pointer.** A source-qualified token — `{source}:{ref}`, `{source}#{ref}`, or the item's own URL —
   resolves through the registry exactly as `WorkSourceRegistry.resolve` does today. A *bare* reference (no source
   prefix: `212`, `BLZ-412`) resolves only when a project is already known: it is offered to each source linked to that
   project whose link's narrowing accepts it (a `repository` narrowing accepts a bare number; a `jira-project` narrowing
   accepts `KEY-n` for its own key). Exactly one acceptor resolves it; none is `422` naming the linked sources; more
   than one is `422` naming each candidate and asking for the source prefix. Source resolution never falls through to
   "first match wins": a qualified token that more than one binding claims is the same `422`.
3. **The project, if not named.** Every pointer's source is looked up among the tenant's links. When all of them are
   linked to exactly one and the same project, that project is used. Otherwise — a source linked to several projects, or
   sources whose single projects differ — the request is `422`, naming the candidate projects.
4. **Every source linked.** Each pointer's source must be linked to the resolved project (`422` naming the unlinked
   source). Linking is how a project declares where it draws from; ingest never links implicitly.
5. **One live holder.** `require_no_live_holder` runs per pointer, unchanged (`409` with `ChunkIngestConflict`). A work
   ref is held by at most one live chunk, so an item sits in at most one project at a time; once its holder is terminal
   it may be ingested again, into any project.

The chunk is minted by `mint_chunk` with the resolved `project_id`, and `chunks.project_id` is never written again.

## Pointers name a source

`chunk_work_refs`, `work_items`, `work_item_closures`, and every other table that stores a work ref keep the
`{source, ref}` shape, keyed by the source's name — which `epic:live-config` makes immutable
([records.md](../../../../delivered/live-config/spec/records.md)) — so a pointer never comes to mean another source.
Projects add nothing to a pointer: the project is the chunk's.

## Grouping

`GroupService.group` (`hub/domain/queue.py`) refuses to fold a chunk into a survivor of a different project: a new
`ChunksInDifferentProjects` error, mapped to `409` beside `ChunkNotGroupable`. A fold within one project is unchanged.

## Closing and annotating

Closing, annotating, and editing a work item go through its pointer's source binding — `WorkSourceRegistry.closer`,
`.annotator`, `.editor` — exactly as today. The chunk's project plays no part: a Jira item ingested into winter is
closed through the tenant's Jira source like any other. A chunk view's artifact branch links resolve through the
repositories its commits resolved to ([delivery.md](./delivery.md)), not through its first work ref's source.

## The built-in source

The built-in `hub` source is one tenant source, linked to every project at the project's creation; its link carries no
narrowing and refuses unlinking. Every path that mints a hub work item and ingests it in one act names the project
explicitly, never relying on resolution step 3:

- `RunService` (`hub/domain/routine_run.py`) ingests a run's item into the routine's own project;
- accepting a garden proposal ingests the minted item into the proposal's project;
- an operator creating a hub item from the board names the shell lens's project, and from the CLI names `--project`.

`hub:<n>` refs stay unique per tenant (`work_item_sequence` is keyed per source, under the tenant key), so a hub item's
ref never collides across projects.
