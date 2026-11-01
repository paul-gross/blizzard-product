# OpenCode 2 specifications

The technical contracts behind `epic:adapters/opencode-2`. Product intent and the decisions behind these contracts begin
at the [epic plan](../index.md); implementation enters here and descends only to the concern it is changing. The 1.x
binding's contracts under [`../../opencode/spec/`](../../opencode/spec/index.md) stay in force for 1.x and are not
edited by this epic.

| Where                                              | Read when                                                                                                                         |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [compatibility.md](./compatibility.md)             | Proving an OpenCode 2 version before implementing or widening its binding                                                         |
| [lines.md](./lines.md)                             | Implementing version lines: binding dispatch, the `@` selector, store isolation, admission                                        |
| [execution.md](./execution.md)                     | Implementing the private server, session lifecycle, prompts, permissions, plugin, or takeover                                     |
| [transcripts.md](./transcripts.md)                 | Implementing 2.x transcript reads, normalization, context, usage, cost, or child-agent records                                    |
| [hardening.md](./hardening.md)                     | Implementing interruption, upgrade detection, health, selftest, analytics, verification, the mock OpenCode 2, or the release gate |
| [harness-selection.md](../../harness-selection.md) | Changing how runners advertise harnesses or how graphs, chunks, and sessions constrain them                                       |
