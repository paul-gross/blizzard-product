# Multi-harness execution

The operator sees a runner, not a Claude machine or an OpenCode machine. A runner may configure both harnesses, prove
both healthy, and advertise both to the hub. The hub uses those capabilities when it offers work; the runner resolves an
authored acceptable set to one concrete adapter when it mints a session. Selection lives above the adapters because
policy is Blizzard's, while each adapter remains a translator of one external harness.

Harness intent belongs to a session lineage. A graph that needs OpenCode for building and Claude Code for review
declares separate sessions and points the nodes at them. A graph content with either declares an ordered acceptable set;
the first capability the holding runner supports wins. Once minted, the session records that harness, and every resume,
judgement, answer, nudge, and takeover returns to the same adapter without consulting defaults again.

Sticky runner tenure remains intact. A runner is eligible to acquire a chunk only when its advertised capabilities can
satisfy every runner-executed session requirement reachable from the chunk's current node. This conservative admission
is what lets a chunk cross from an OpenCode build node to a Claude Code review node without being stranded halfway
through its graph. A chunk no available runner can carry simply holds its place and waits; work that cannot be claimed
stops flowing, and this slice is content to let it.

What does need deciding is the runner's own conduct at the queue. A runner asks the hub for work it can actually run,
and says whether it would rather hold at the first chunk it cannot work or pass over it — a machine bought to run
OpenCode work may be meant to wait for it, while a mixed fleet is better served by taking the next chunk it can finish.
The hub answers with an entry or with nothing. It never records or reports why it skipped one, because a fleet that
explains its own stalls is a larger idea than this slice needs.

For OpenCode, the usable worker includes headless spawn and resume, two-phase judgement, answer delivery, nudges,
takeover, heartbeat and session-end behavior, permissions suitable for unattended execution, model and effort
resolution, and completed-turn usage and cost. The [harness selection specification](./spec/harness-selection.md) owns
the cross-entity contract; the [OpenCode execution specification](./spec/execution.md) owns the binding.
