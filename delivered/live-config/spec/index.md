# Live configuration specifications

The technical contracts behind `epic:live-config`. Product intent begins at the [epic plan](../index.md); implementation
enters at the concern below. The configuration screens these contracts serve are the plan's
[mockups](../artifacts/configuration-desktop.html).

| Where                            | Read when                                                                                                                 |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| [records.md](./records.md)       | Shaping a configured record — the work-source and repository tables, revisions, retirement, and the change log            |
| [secrets.md](./secrets.md)       | Building the secret store — write-only values, encryption at rest, the hub key, references by name, rotation              |
| [api.md](./api.md)               | Adding or reviewing any configured surface — the shared verb set, patch meaning, file ingestion, apply, and the CLI       |
| [runtime.md](./runtime.md)       | Wiring a reader of configuration — read on use, caching by revision, what a change does to work in flight                 |
| [carry-over.md](./carry-over.md) | Building the first-start import from `blizzard-hub.toml` and `BZ_FORGE_*`, and the refusal to start while old keys remain |
