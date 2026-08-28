# Adapter specifications

The technical contracts behind `epic:adapters`. Product intent begins at the [epic plan](../index.md); implementation
enters here and descends only to the concern it is changing.

| Where                                          | Read when                                                                                   |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------- |
| [compatibility.md](./compatibility.md)         | Proving an OpenCode version before implementing or upgrading its binding                    |
| [harness-selection.md](./harness-selection.md) | Changing how runners advertise harnesses or how graphs, chunks, and sessions constrain them |
| [execution.md](./execution.md)                 | Implementing OpenCode process, session, hook, output, permission, or takeover behavior      |
| [transcripts.md](./transcripts.md)             | Implementing OpenCode transcript reads, normalization, context telemetry, or usage recovery |
| [hardening.md](./hardening.md)                 | Implementing capability health, conformance, analytics recognition, or release verification |
