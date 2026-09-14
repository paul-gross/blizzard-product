---
epic: advanced-delivery
refinement: refined
slices:
  - name: full
    status: horizon
---

# Plan — `epic:advanced-delivery`

Landing finished work is where a whole fleet converges on one branch, and the habit blizzard lands it with is a single
one: a policy script the hub runs, one chunk at a time across the entire fleet, against one forge. That habit suits a
fleet of a few agents working one project on GitHub. It strains the moment a shop wants something else — a human's hand
on every merge, a GitLab merge train, a chunk whose work spans repositories on two forges — and it strains hardest when
many agents finish at once, because every landing moves the branch under each chunk still waiting, and each of those
goes back to an agent to rebase work that was already right.

This plan is early. The shape below is understood and worth keeping; most of the decisions it forces are not yet made,
and the open questions at the end are the real work of promoting it.

## The shape

Delivery, stripped of git, is three things: a deterministic step the hub executes, exclusive access to a shared
resource, and an act followed by patient observation until the world settles. Deployment is the same three things over a
different resource, which is why this epic and `epic:advanced-deployment` share one foundation rather than each building
its own.

- **A step scoped to a named resource.** A hub step declares what it holds — `repo:<name>/<branch>`, `env:<name>` — and
  exclusivity is per resource. The single fleet-wide slot becomes the degenerate case in which every step holds the same
  resource, and ordering on a resource becomes an explicit queue rather than whoever reaches the slot first.
- **Chunks submit; resource flows land.** A chunk's graph does what belongs to one chunk — building it, and resolving
  its conflicts, which need its own environments and agents. Its deliver node submits the landed candidate to the
  branch's flow and waits for a verdict. The flow does what spans many chunks: batching, ordering, landing, and
  reporting the outcome back to every chunk it carried. Today's fast-forward landing is a flow with a batch of one.
- **One event joins delivery to what follows it.** A branch flow emits that it landed a set of commits for a set of
  chunks; environment flows consume it. Flows chain, so a delivery train is a train in the literal sense.

## What to build

- **A landing seam, beside the work-source seam.** A forge binding already reads, labels, closes, and edits work items;
  it gains a landing half that reads and advances refs, opens, reads, updates, and merges changes, and joins native
  queues. Each forge normalizes its own vocabulary into one set of change states, so a policy never reads GitHub's words
  or GitLab's. What a binding can do is declared, and a graph asking a forge for a policy it cannot honor is refused
  when the graph is minted.
- **Landing policies with one outcome vocabulary.** Fast-forward, merge once CI is green, park for a human and take over
  once it merges, and join the forge's queue are policies, chosen per repository with a project default. Every policy
  reports the same outcomes — landed, pending, conflict, failure, awaiting a human — so a graph written for one runs
  unchanged under another.
- **Repositories bound through their project.** A chunk's commits resolve to repositories in its project, each bound to
  its forge, so a chunk spanning forges lands through each repository's own binding. Its combined outcome is landed only
  when every repository has landed, a declared order puts a library before its consumer, and a partial land is either
  accepted and reconciled, as today, or held until every repository is ready.
- **Merge trains.** Mechanical rebasing comes first, waking an agent only for a textual conflict no machine can settle.
  Beyond it, a flow batches the ready work onto the base and proves the combination once, splitting a failing batch to
  find the chunk responsible, or builds speculatively along the queue; where the forge offers its own merge train or
  merge queue, the flow may hand ordering to it instead. A chunk whose work cannot join the train is ejected back to its
  own graph to resolve, and the train moves on without it.
- **The combination is what gets proven.** Two changes that merge cleanly can still break each other, so whatever policy
  lands work, the tree that reaches the base is the tree that was verified.

## Open questions

- Whether a resource flow is itself a chunk — minted by the system on a delivery graph, carrying the chunks it lands as
  its work — so it inherits tenure, facts, the board, asks, and transcripts, or whether it is a new kind of subject.
- Whether a landing script reaches the forge through the hub, by a callback in the manner of the marker channel, so no
  script holds a token and there is one adapter implementation, or through a self-contained adapter it carries.
- Whether the landing seam grows out of today's work-source bindings, or a forge binding becomes its own configuration
  that work sources and repositories both reference.
- Whether a repository may belong to more than one project.
- What "done" means for a chunk: landed, or live. If live, a chunk waiting on its deploy must release its environments
  before it waits.
- How a merge made by a human is noticed at scale — polling every parked change, or forge events that wake the hub — and
  what the fleet does when that signal is late or missed.
- Whether the queue should avoid conflicts before they happen, keeping chunks likely to touch the same ground from
  running side by side, and whether that belongs here or to the queue.
