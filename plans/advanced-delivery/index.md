---
epic: advanced-delivery
refinement: refined
slices:
  - name: actions
    status: horizon
    plan: ./actions/index.md
  - name: forges
    status: horizon
    plan: ./forges/index.md
  - name: flows
    status: horizon
    plan: ./flows/index.md
  - name: policies
    status: horizon
    plan: ./policies/index.md
  - name: trains
    status: horizon
    plan: ./trains/index.md
---

# Plan — `epic:advanced-delivery`

Landing finished work is where a whole fleet converges on one branch, and the habit blizzard lands it with is a single
one: a script the hub starts as a process of its own, one chunk at a time across a whole tenant, against one forge. That
habit suits a fleet of a few agents working one project on GitHub. It strains the moment a shop wants something else — a
human's hand on every merge, a GitLab merge train, a chunk whose work spans repositories on two forges — and it strains
hardest when many agents finish at once, because every landing moves the branch under each chunk still waiting, and each
of those goes back to an agent to rebase work that was already right.

## The shape

Delivery, stripped of git, is three things: a deterministic step the hub executes, exclusive access to a shared
resource, and an act followed by patient observation until the world settles. Deployment is the same three things over a
different resource, which is why this epic and `epic:advanced-deployment` share one foundation rather than each building
its own.

- **A hub step is one of the hub's own actions.** A graph names what a hub step does, never a command, and the hub does
  it in its own code, a short pass at a time. Nothing a tenant writes runs with the hub's reach, and the hub can take
  many steps at once and decide for itself what comes next.
- **A step scoped to a named resource.** A hub step declares what it holds — `repo:<repository>/<branch>`, `env:<name>`
  — and exclusivity is per resource. The one slot every step in a tenant shares today becomes the degenerate case in
  which every step holds the same resource, and ordering on a resource becomes an explicit queue rather than whoever
  reaches the slot first.
- **Chunks submit; resource flows land.** A chunk's graph does what belongs to one chunk — building it, and resolving
  its conflicts, which need its own environments and agents. Its deliver node submits the landed candidate to the
  branch's flow and waits for a verdict. The flow does what spans many chunks: batching, ordering, landing, and
  reporting the outcome back to every chunk it carried. Today's fast-forward landing is a flow with a batch of one.
- **A flow lives at the hub, as a subject of its own.** It is named by the resource it holds, keeps an append-only
  record of what it admitted, proved, landed, and ejected, and shows on the board as a lane per resource. It is not a
  chunk: it carries several projects' work, changes its contents in flight, and needs no runner. A chunk waiting on a
  flow is `delivering`, and the flow's verdict reaches it as a fact.
- **One event joins delivery to what follows it.** A branch flow emits that it landed a set of commits for a set of
  chunks; environment flows consume it. Flows chain, so a delivery train is a train in the literal sense. A chunk's
  delivery ends at landing: what happens after it is `epic:advanced-deployment`'s, and never holds the chunk or its
  environments open.

## What to build

- **A forge, declared once per tenant.** A forge is a tenant record for one server: its kind, its API and web addresses,
  how the hub authenticates to it, the rate budget everything calling it shares, and the capabilities the server itself
  offers, such as whether its tier runs merge trains. A repository names the forge it lives on, and so does a work
  source whose provider is a forge; a tracker that is not one names none. What belongs to one use stays on that record:
  its own credential, so landing and reading work can hold different keys, and its own settings — a repository's base
  branch, landing policy, and the capabilities it switches on, a work source's opt-in to forge-status annotation.
- **A landing seam, beside the work-source seam.** A forge binding already reads, labels, closes, and edits work items;
  it gains a landing half that reads and advances refs, opens, reads, updates, and merges changes, and joins native
  queues. Each forge normalizes its own vocabulary into one set of change states, so a policy never reads GitHub's words
  or GitLab's. Landing is the hub's own action, so no credential leaves the hub, and it speaks this seam rather than any
  one forge's API, so there is one adapter per forge and it lives at the hub. An action names the capabilities it needs,
  and a repository offers what its forge has, narrowed to what the repository switches on; a missing capability takes
  the fallback the action declares for it — the flow's own train for a forge with no merge queue — or, with none
  declared, is refused. A graph is shared across the tenant and cannot know where it will land, so the check runs when a
  repository's policy is set and again when a chunk's repositories resolve at landing.
- **Landing policies with one outcome vocabulary.** Fast-forward, merge once CI is green, park for a human and take over
  once it merges, and join the forge's queue are policies. A policy describes the repository, so it sits on the tenant's
  repository record, where `epic:projects` left room for it; a default on a project's link to the repository comes only
  if one earns its place. Projects that link one repository share its one queue, and the repository's policy is the one
  that holds there. A parked change is noticed merged by polling, every two minutes at the most often; hearing of the
  merge from the forge's webhook instead is not this epic's. Every policy reports the same outcomes — landed, pending,
  conflict, failure, awaiting a human — so a graph written for one runs unchanged under another.
- **Landing across repositories and forges.** `epic:projects` already confines a chunk to the repositories its project
  links, and each of a chunk's repositories already lands against its own record — its own server, owner, base branch,
  and credential — so a chunk spanning two GitHub servers lands today. What remains is landing across kinds of forge and
  across policies, each repository through its own binding. The combined outcome is landed only when every repository
  has landed, a declared order puts a library before its consumer, and a partial land is either accepted and reconciled,
  as today, or held until every repository is ready.
- **Merge trains.** Mechanical rebasing comes first, waking an agent only for a textual conflict no machine can settle.
  Beyond it, a flow batches the ready work onto the base and proves the combination once, splitting a failing batch to
  find the chunk responsible, or builds speculatively along the queue; where the forge offers its own merge train or
  merge queue, the flow may hand ordering to it instead. A chunk whose work cannot join the train is ejected back to its
  own graph to resolve, and the train moves on without it.
- **The combination is what gets proven.** Two changes that merge cleanly can still break each other, so whatever policy
  lands work, the tree that reaches the base is the tree that was verified.

## Slices

| Slice                             | Carries                                                                                                                                                                                                |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [`actions`](./actions/index.md)   | Hub steps as the hub's own actions taken a pass at a time, the packaged scripts carried over as actions, and `run:` retired                                                                            |
| [`forges`](./forges/index.md)     | The forge record, repositories and work sources naming their forge, and the landing seam with its capabilities, which `land` speaks                                                                    |
| [`flows`](./flows/index.md)       | Steps scoped to a named resource, the flow as a subject at the hub, a chunk submitting and awaiting a verdict, and today's landing as a flow with a batch of one                                       |
| [`policies`](./policies/index.md) | Three landing policies on the repository record, noticing a person's merge by polling within the forge's shared rate budget, GitLab as a second forge kind, and landing across repositories and forges |
| [`trains`](./trains/index.md)     | Mechanical updating, batches proven once with the combination landed, capabilities a repository switches on, the forge's own queue as a fourth policy, and ejecting what cannot join                   |
