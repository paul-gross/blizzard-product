# Projects hub specifications

The technical contracts behind the hub slice of `epic:projects`. Product intent begins at the [slice plan](../index.md);
implementation enters at the concern below. The tenant key, and the store seam that scopes every read to a tenant and
narrows it to a project on request, are owned by the multi-tenancy
[store contract](../../../multi-tenancy/hub/spec/store.md). Work-source and repository records, their revisions, and the
secrets they name are owned by `epic:live-config`'s [records](../../../live-config/spec/records.md) and
[secrets](../../../live-config/spec/secrets.md) contracts. This spec adds the project to them; it redefines neither.

| Where                              | Read when                                                                                                     |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| [model.md](./model.md)             | Shaping the store — the project record, source links, which tables carry `project_id`, uniqueness, retirement |
| [ingest.md](./ingest.md)           | Building ingest — resolving a token and a project, the refusals, grouping, closing, the built-in source       |
| [delivery.md](./delivery.md)       | Landing a chunk — repository links, landing only where the project links, and the `run:` environment          |
| [eligibility.md](./eligibility.md) | Matching runners to work — the served-projects declaration, peek and claim filters, wire compatibility        |
| [surfaces.md](./surfaces.md)       | Building the API, CLI, and web surfaces — project parameters, the shell lens, each view's semantics           |
| [carry-over.md](./carry-over.md)   | Writing the migration — how a single-project installation becomes one project without re-ingest               |
