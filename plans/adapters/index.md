---
epic: adapters
refinement: refined
slices:
  - name: claude-code
    status: delivered
    plan: ./claude-code/index.md
  - name: opencode
    status: horizon
    plan: ./opencode/index.md
  - name: codex
    status: horizon
    plan: ./codex/index.md
---

# Plan — `epic:adapters`

An operator should be able to choose the agent that fits the work rather than accept whichever coding harness Blizzard
happened to support first. Today that choice stops at the model: every worker still enters through Claude Code, so a
model available only through another harness is outside the fleet no matter how capable or economical it proves.

The harness becomes a resolved execution capability rather than an identity of the runner. One runner may hold several
harnesses, advertise the ones it can actually operate, and use different ones for different session lineages in the same
chunk. A graph may constrain a lineage to one harness or to an ordered set that accepts several; a chunk may supply the
default where the graph is silent; and the runner supplies the final default. Order makes the choice reproducible: the
first supported acceptable harness wins, never a random draw and never an undeclared substitute.

The epic lands one harness at a time. The delivered Claude Code slice is the foundation rather than scope to rebuild.
OpenCode is the second live binding and therefore the test of whether the seam is real. Codex follows the same contract
after two occupants have forced every Claude-shaped assumption into the binding that owns it.

| Where                                          | Read when                                                                                                                             |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| [claude-code/](./claude-code/index.md)         | Looking up the delivered adapter foundation every later harness extends                                                               |
| [opencode/](./opencode/index.md)               | Planning or implementing OpenCode as the second harness, and harness selection                                                        |
| [codex/](./codex/index.md)                     | Reasoning about Codex as the third harness                                                                                            |
| [harness-selection.md](./harness-selection.md) | Changing how runners advertise harnesses or how graphs, chunks, and sessions constrain them — the contract every harness slice shares |

## Boundaries

This epic does not make one harness impersonate another. A capability with no honest equivalent in a harness stays
absent and visible as absent; the platform does not invent a context measurement, reinterpret a compaction threshold, or
weaken a permission deny to claim parity. It does not carry a conversation across harnesses either: two harnesses may
work the same chunk and the same environments, but they meet through durable artifacts and the worktree, never by asking
one harness to resume another's session.

Automatic cross-harness retry is a later policy, not an accidental consequence of adding a second adapter. The OpenCode
slice keeps a retry on the harness that began the attempt and escalates when that harness is unavailable. A future
policy may choose another member of an authored acceptable set, but only while minting a fresh session and only as an
explicit, recorded decision.
