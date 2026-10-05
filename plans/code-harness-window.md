---
epic: code-harness-window
refinement: scaffolded
slices:
  - name: agent-observer
    status: horizon
  - name: swarm-view
    status: horizon
---

# Plan — `epic:code-harness-window`

An operator watching a chunk on the board sees one card, one session, one node. For most chunks that is the truth. For a
chunk running a swarm it is barely a summary: behind that one card, a single session may be running dozens of agents at
once, each with a context of its own, some of them quietly growing past the point where they still think clearly. The
architectural sweep ran about four hundred and fifty agents through one orchestrating session, and its first generation
grew to more than half a million tokens of context before anyone noticed — the operator caught it by asking, and the
orchestrator answered by building a watcher of its own that polled every agent each minute and raised a hand at a
threshold. That watcher was the right idea, built in the wrong place: it lived inside one run and died with it.

This epic gives the operator a window into the coding harness while it works: what context the session is carrying,
which agents it has running, and how each of them is doing. The window keeps the shape blizzard already holds to. The
hub cannot reach into a runner, and should not; the runner is the one that looks, and what it sees travels up as facts
like everything else it reports, so the board renders a swarm the same way it renders a lease.

## The agent observer

The first slice is the observer itself: a lane on the runner that reads each running session's harness transcripts on an
interval and reports what it finds. For each running chunk it reports how many agents are live, how much context each
one is carrying, which have crossed the operator's warn line, and how many tokens the session and its agents have spent
so far. The board shows it where the operator already looks — a small reading on the chunk's card, something like twelve
agents live, the heaviest at two hundred thirty thousand, three over the line — and the line crossing arrives as an
event the way a session's own context warning does today.

Much of the ground is already laid. The runner already samples a running session's context against a warn line and
reports the first crossing, and it already reads the private conversations of the subagents a session spawns when it
ships transcripts to the hub. The observer widens the first to cover the second.

Its spend reading matters beyond the window. A swarm spends its whole budget inside a single node, where today's spend
controls cannot see it; the observer's running count is what would let a ceiling reach inside the node at all, which is
the open question the `epic-dwf` slice of `epic:chokepoint` leaves for this one to answer.

## The swarm view

The second slice is the larger window the first one opens onto: the swarm's structure rather than its vital signs —
which workflow runs a session has launched, the phase each is in, which agents belong to which run, and the session's
own running record of where the work stands. It is deliberately second. The observer answers whether the swarm is
healthy; the swarm view answers what it is doing, and it is worth building only once the first has shown which questions
operators actually ask.

## Open questions

- Where a dynamic workflow's agents leave their transcripts. The runner reads the subagents a session spawns directly;
  whether agents launched inside a background workflow run land in the same place is the first thing to establish, and
  the answer decides how much of the observer is reading and how much is new.
- Which harnesses can be observed. Each harness keeps its transcripts its own way, so each one answers for its own
  agents behind the harness seam. A harness that cannot run a swarm has little to report beyond its main session.
- Whether the observer only reports or may also act — stopping an agent past the line, or asking the session to hand it
  off. Reporting comes first; acting reaches into `epic:steering`'s territory and should be decided there.
