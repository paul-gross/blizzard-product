---
initiative: advanced-delivery
refinement: refined
epics:
  - actions
  - forges
  - flows
  - policies
  - gitlab
  - trains
---

# Plan — `initiative:advanced-delivery`

Landing finished work is where a whole fleet converges on one branch, and blizzard lands it with a single habit: open a
pull request on GitHub, wait for its checks, merge it — one chunk at a time across a whole tenant, through a script the
hub starts as a process of its own. That habit suits a fleet of a few agents working one project on GitHub. It strains
the moment a shop wants something else — a human's hand on every merge, a main branch fast-forwarded with no pull
request at all, a GitLab project, a chunk whose work spans repositories on two kinds of forge — and it strains hardest
when many agents finish at once. Every landing moves the branch under each chunk still waiting. The hub brings each of
those pull requests up to the new base on its own, without waking an agent, but every one of them then runs its checks
again, and on a busy night the line grows faster than it drains. A chunk that waits long enough through those re-runs is
taken for stuck and sent back to an agent to repair work that was already right.

## The shape

Delivery, stripped of git, is three things: a deterministic step the hub executes, exclusive access to a shared
resource, and an act followed by patient observation until the world settles.

- **A hub step is one of the hub's own actions.** A graph names what a hub step does, never a command, and the hub does
  it in its own code, a short pass at a time. An action works only from what it was handed and reports what it did, and
  the hub records each thing it did before it goes on, so nothing a tenant writes runs with the hub's reach, the hub can
  take many steps at once, and a crash mid-step leaves a record the next pass can trust.
- **A step scoped to a named resource.** A hub step declares what it holds — `repo:<repository>/<branch>` — and
  exclusivity is per resource. The one slot every step in a tenant shares today becomes the degenerate case in which
  every step holds the same resource, and ordering on a resource becomes an explicit queue rather than whoever reaches
  the slot first.
- **Chunks submit; flows land.** A chunk's graph does what belongs to one chunk — building it, and resolving its
  conflicts, which need its own environments and agents. Its deliver node submits the landed candidate to the branch's
  flow and waits for a verdict. The flow does what spans many chunks: batching, ordering, landing, and reporting the
  outcome back to every chunk it carried. Today's landing is a flow with a batch of one.
- **A flow lives at the hub, as a subject of its own.** It is named by the branch it holds, keeps an append-only record
  of what it admitted, proved, landed, and ejected, and shows on the board as a lane per branch. It is not a chunk: it
  carries several projects' work, changes its contents in flight, and needs no runner. A chunk waiting on a flow is
  `delivering`, and the flow's verdict reaches it as a fact.
- **Landing ends delivery, and says so once.** A flow records each landing as one event — the commits it landed, the
  chunks they belonged to, and each chunk's project — and whatever follows landing starts from that event. A chunk's
  delivery ends there, and never holds the chunk or its environments open past it.
- **Patience where waiting is expected.** A wait the hub expects — a train proving its batch, a forge's queue moving, a
  forge asking the hub to slow down — is given hours rather than the half hour a landing has today before anyone calls
  it stuck, and a wait no machine will end, such as a parked change a person closed without merging, goes to a person
  rather than an agent.

## What to build

- **A forge, declared once per tenant.** A forge is a tenant record for one server: its kind and its API and web
  addresses. A repository names the forge it lives on, and so does a work source whose provider is a forge; a tracker
  that is not one names none. What belongs to one use stays on that record: its own credential, so landing and reading
  work can hold different keys, and its own settings — a repository's base branch and landing policy, a work source's
  opt-in to forge-status annotation. What a forge can do comes from its kind, narrowed for each repository by what the
  hub finds when it asks — whether a merge queue runs on that repository's base, whether the hub may advance it directly
  — and every caller of a forge shares the rate budget each of its keys has, as the forge reports it.
- **A landing seam, beside the work-source seam.** A forge's work-source binding already reads, labels, and closes work
  items; landing gains a seam of its own that reads and advances refs, opens, reads, updates, and merges changes, reads
  and re-runs checks, and joins native queues, offered in parts so each caller holds only the part it uses. Each forge
  normalizes its own vocabulary into one set of change and check states, so a policy never reads GitHub's words or
  GitLab's. A forge's momentary refusals — a race on the base, a rate limit, an outage — are waits, never failures, and
  a refusal no agent can repair, such as a rejected credential or a repository that is not there, goes to a person.
  Landing is the hub's own action, so no credential leaves the hub, and it speaks this seam rather than any one forge's
  API, so there is one adapter per forge kind and it lives at the hub. GitHub's is the first and, until the `gitlab`
  epic, the only one: the seam is drawn so that any forge fits behind it, and GitLab is the evidence that one does. A
  policy names the capabilities it needs; a missing one takes the fallback the policy declares — merging once checks
  pass, for a forge queue the repository does not have — or, with none declared, is refused. A graph is shared across
  the tenant and cannot know where it will land, so the check runs when a repository's policy is set and again when a
  chunk's repositories resolve at landing.
- **Landing policies with one outcome vocabulary.** Fast-forward, merge once checks pass, park for a human and take over
  once it merges, and join the forge's queue are policies. A policy describes the repository, so it sits on the tenant's
  repository record, where `initiative:projects` left room for it; a default on a project's link to the repository comes
  only if one earns its place. Projects that link one repository share its one queue, and the repository's policy is the
  one that holds there. A parked change is noticed merged by polling, every two minutes at the most often; hearing of
  the merge from the forge's webhook instead is not this initiative's. Every policy answers in the same verdicts —
  landed, conflict, failure, inherited-failure — and waits in the same two ways, pending while checks run and awaiting a
  person while a parked change is unmerged, so a graph written for one runs unchanged under another.
- **Landing across repositories and forges.** `initiative:projects` already confines a chunk to the repositories its
  project links, and each of a chunk's repositories already lands against its own record — its own server, owner, base
  branch, and credential — so a chunk spanning two GitHub servers lands today. What remains is landing across policies,
  each repository through its own binding, in an order its project sets, and, once GitLab joins, across kinds of forge.
  Each project ranks the repositories it links, by dragging them into place: a library ranked above its consumer lands
  first, and a repository left unranked lands after every ranked one, the unranked in order of their slugs. Two projects
  linking the same repositories may rank them differently, since a chunk belongs to one project and lands by its order.
  A chunk lands into none of the repositories the hub lands itself until every one of them is ready, then lands them in
  its project's order, one waiting behind a parked repository until the person merges. Landing across forges can never
  be a single act, so should one repository refuse after another has landed, the landed one stands and the chunk
  reconciles what remains, as it does today.
- **Merge trains.** A pull request behind its base is already brought up to it without an agent; what a busy branch
  lacks is a way to stop re-proving every chunk each time another lands. A flow batches the ready work onto the base and
  proves the combination once, splitting a failing batch to find the chunk responsible; where the forge offers its own
  merge train or merge queue, the flow may hand ordering to it instead. An agent is woken only for a textual conflict no
  machine can settle. A chunk whose work cannot join the train is ejected back to its own graph to resolve, and the
  train moves on without it.
- **Each policy says what it proves.** Two changes that merge cleanly can still break each other. Where the hub moves
  the branch itself — fast-forward, and the flow's own train — the tree that reaches the base is the tree that was
  verified. Merging once checks pass proves the change's own branch, and the merged tree only where the forge itself
  requires a change to be up to date with its base before it merges; a parked change lands whatever the person merged.
  No policy claims more than it keeps.

## Epics

| Epic                              | Carries                                                                                                                                                                                                   |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`actions`](./actions/index.md)   | Hub steps as the hub's own actions, taken a pass at a time from only what they are handed, the packaged scripts carried over as actions, and `run:` retired                                               |
| [`forges`](./forges/index.md)     | The forge record, repositories and work sources naming their forge, and the landing seam — in parts, with its capabilities and its rule for which refusals wait — which `land` speaks                     |
| [`flows`](./flows/index.md)       | Steps scoped to a named resource, the flow as a subject at the hub, a chunk submitting and awaiting a verdict, and today's landing as a flow with a batch of one                                          |
| [`policies`](./policies/index.md) | Three landing policies on the repository record, each project's landing order, noticing a person's merge by polling within the forge's shared rate budget, and landing across repositories, all on GitHub |
| [`gitlab`](./gitlab/index.md)     | GitLab as a second forge kind behind the same seam — merge requests, pipelines, nested groups, a rebase the hub asked for — and landing across kinds of forge                                             |
| [`trains`](./trains/index.md)     | Batches proven once with the combination landed, capabilities probed for each repository, the forge's own queue as a fourth policy, and ejecting what cannot join                                         |
