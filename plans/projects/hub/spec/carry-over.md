# Carry-over contract

An installation that predates this slice becomes a tenant holding one project, in one migration, with nothing
re-ingested and nothing that runs reconfigured. It runs after the multi-tenancy carry-over has placed every row in the
default tenant, and after `epic:live-config`'s carry-over has turned the hub's file and environment into work-source,
repository, and secret records ([carry-over.md](../../../live-config/spec/carry-over.md)).

## The migration

One manual migration (`bzh:manual-migrations`), frozen once released (`bzh:frozen-revisions`), inside the portable
surface (`bzh:sql-portable`). Within each tenant that holds any state, it:

1. **Creates the project.** A fresh `proj_<ulid>` id, named `default`; description empty; revision 1; a change fact
   attributed to `migration`. The operator renames it afterwards like any other project.
2. **Links every work source.** One `project_source_links` row per work-source record, unnarrowed, including the
   built-in `hub` source.
3. **Links every repository.** One `project_repository_links` row per repository record, so every repository the hub
   delivers into today stays a repository its chunks may land in.
4. **Stamps the project-owned tables.** `chunks`, `scopes`, `routines`, `findings`, `finding_sets`, `garden_proposals`,
   and `work_item_runs` get the project's id, and `work_items` rows carrying a routine pair get it too. Scope and
   routine uniqueness narrows from the tenant to the project ([model.md](./model.md) §Scope and routine identity); no
   slug or routine name changes, because one project cannot collide with itself.
5. **Records what every runner serves.** Every `runner_registrations` row's `projects` becomes the one project. A runner
   that re-registers without a declaration keeps it ([eligibility.md](./eligibility.md)), so the fleet keeps claiming
   exactly what it claimed before.

## What does not change

No chunk is re-minted, no work ref is rewritten, no graph pin moves, and no fact is edited: the project is stamped onto
rows that had no grouping, and every append-only table stays append-only. The built-in source's `work_item_sequence`
continues from where it stood. The board, CLI, and API behave as before for a caller that names no project, because the
tenant then holds exactly one.

## Proof

The invariant checker's project rules ([model.md](./model.md) §Invariants) pass on a carried-over store. A fixture store
captured before the migration, carried over, serves the same `GET /api/chunks`, queue order, and matched peek for each
registered runner as it did before, with `project` added to each chunk.
