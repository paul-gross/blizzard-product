# Plan — `epic:adapters`

An operator should be able to choose the agent that fits the work rather than accept whichever coding harness Blizzard
happened to support first. Today that choice stops at the model: every worker still enters through Claude Code, so a
model newly available through OpenCode is outside the fleet no matter how capable or economical it proves. The immediate
proving case is ChatGPT 5.6 Luna at `max` effort through OpenCode, where early use shows the appetite of a practical
build agent at a cost worth comparing against the incumbent.

The harness becomes a resolved execution capability rather than an identity of the runner. One runner may hold several
harnesses, advertise the ones it can actually operate, and use different ones for different session lineages in the same
chunk. A graph may constrain a lineage to Claude Code, OpenCode, or an ordered set that accepts either; a chunk may
supply the default where the graph is silent; and the runner supplies the final default. Order makes the choice
reproducible: the first supported acceptable harness wins, never a random draw and never an undeclared substitute.

OpenCode is the second live binding and therefore the test of whether the seam is real. Codex follows the same contract
after two occupants have forced every Claude-shaped assumption into the binding that owns it. The delivered Claude Code
slice remains the foundation rather than scope to rebuild.

| Where                                  | Read when                                                                       |
| -------------------------------------- | ------------------------------------------------------------------------------- |
| [claude-code.md](./claude-code.md)     | Looking up the delivered adapter foundation this breadth slice extends          |
| [compatibility.md](./compatibility.md) | Deciding what OpenCode must prove before production implementation begins       |
| [execution.md](./execution.md)         | Planning the first usable OpenCode worker and multi-harness routing slice       |
| [transcripts.md](./transcripts.md)     | Planning conversation, context, and interrupted-usage parity for OpenCode       |
| [hardening.md](./hardening.md)         | Planning the operational proof that makes OpenCode safe to advertise unattended |
| [spec/](./spec/index.md)               | Implementing any slice or resolving its technical contract                      |

## Boundaries

This epic does not make one harness impersonate another. A capability with no honest OpenCode equivalent stays absent
and visible as absent; the platform does not invent a context measurement, reinterpret a compaction threshold, or weaken
a permission deny to claim parity. It does not carry a conversation across harnesses either: two harnesses may work the
same chunk and the same environments, but they meet through durable artifacts and the worktree, never by asking one
harness to resume another's session.

Automatic cross-harness retry is a later policy, not an accidental consequence of adding a second adapter. The first
breadth slice keeps a retry on the harness that began the attempt and escalates when that harness is unavailable. A
future policy may choose another member of an authored acceptable set, but only while minting a fresh session and only
as an explicit, recorded decision.
