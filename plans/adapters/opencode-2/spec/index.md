# OpenCode 2 specifications

The technical contracts behind `epic:adapters/opencode-2`. Product intent and the decisions behind these contracts begin
at the [epic plan](../index.md); implementation enters here and descends only to the concern it is changing. The 1.x
binding's contracts under [`../../opencode/spec/`](../../opencode/spec/index.md) stay in force for 1.x and are not
edited by this epic.

| Where                                              | Read when                                                                                                     |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| [compatibility.md](./compatibility.md)             | Proving an OpenCode 2 version before implementing or admitting it, or its verification-matrix companion edit  |
| [versions.md](./versions.md)                       | Implementing major versions: dispatch by version, sessions, admission, configuration, store isolation         |
| [execution.md](./execution.md)                     | Implementing the driver, session lifecycle, stopping, permissions, the plugin, models, or takeover            |
| [transcripts.md](./transcripts.md)                 | Implementing 2.x read servers, transcript reads, normalization, context, usage, or cost                       |
| [hardening.md](./hardening.md)                     | Implementing health, selftest, concurrency, analytics, verification, the mock OpenCode 2, or the release gate |
| [harness-selection.md](../../harness-selection.md) | Changing how runners advertise harnesses or how graphs, chunks, and sessions constrain them                   |
