# Plan — `epic:adapters`, OpenCode slice

OpenCode is the second harness behind the adapter seam, and the first that proves the seam is real. The immediate
proving case is ChatGPT 5.6 Luna at `max` effort through OpenCode, where early use shows the appetite of a practical
build agent at a cost worth comparing against the incumbent. The slice makes OpenCode a first-class worker beside Claude
Code — on the same runner, in the same chunk, chosen per session lineage — and builds the harness selection every later
harness will use.

It lands in four steps, each gated on the one before: prove what the real CLI does, make a usable worker of it, bring
its conversations to parity, and harden it until it is safe to advertise unattended.

| Where                                  | Read when                                                                       |
| -------------------------------------- | ------------------------------------------------------------------------------- |
| [compatibility.md](./compatibility.md) | Deciding what OpenCode must prove before production implementation begins       |
| [execution.md](./execution.md)         | Planning the first usable OpenCode worker and multi-harness routing             |
| [transcripts.md](./transcripts.md)     | Planning conversation, context, and interrupted-usage parity for OpenCode       |
| [hardening.md](./hardening.md)         | Planning the operational proof that makes OpenCode safe to advertise unattended |
| [spec/](./spec/index.md)               | Implementing any part of the slice or resolving its technical contract          |
