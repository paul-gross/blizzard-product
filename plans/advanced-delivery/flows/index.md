# Plan — `epic:advanced-delivery`, flows slice

The hub runs one step at a time for a whole tenant. Landing a chunk into `blizzard`, recording a garden run's findings,
and landing an unrelated chunk into a different repository all wait on the same slot, and the order in which they get it
is whoever happens to ask first on the next tick. The slot was the simplest way to keep two landings off one branch, and
it does that, but it keeps everything else apart too: a tenant with three projects lands their work single file, and an
operator watching a chunk sit in `delivering` cannot see what it is waiting behind or how long the line is.

This slice gives landing the shape the rest of the epic builds on. A step says which resource it needs — the branch it
lands on — and waits only for that; two landings into different repositories run side by side, and a step that touches
no repository waits for nothing. Landing on a branch becomes a flow: a subject at the hub, one per branch, that chunks
submit their work to and that answers each of them with a verdict. In this slice a flow lands one chunk at a time, the
way the slot does today, so nothing about how work lands changes; what changes is that the waiting has a place, an
order, and a record.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

- **A step holds a named resource.** A hub step that lands into a repository holds `repo:<repository>/<branch>` for each
  repository its chunk's commits resolve to, and two steps conflict only when they hold the same resource. A step that
  lands nowhere — recording a garden delivery, recording review findings — holds nothing and runs whenever its chunk
  reaches it. A step holding several resources takes them together or not at all, so two multi-repository chunks never
  wait on each other with one branch apiece.
- **A flow per branch, at the hub.** A flow is its own subject, named by the resource it holds, and never a chunk. It
  keeps an append-only record of what it admitted, landed, and turned back, in what order, and why, so the question
  "what happened on `master` last night" has one place to be answered.
- **Chunks submit; the flow answers.** A chunk standing at a hub step is in line on the flow of each branch it lands on
  and waits, `delivering`, with its runner keeping its environments as today. The flow gives submissions their turns in
  the order they arrived and lands each by taking the chunk's own delivery action. A chunk waiting on its checks gives
  up its turn and keeps its place, so a branch is never held through a wait on CI; the hub starts each turn itself, so a
  chunk whose runner has gone still lands. Its verdict — landed, conflict, failure — reaches the chunk as a fact and
  routes it along the edge its graph authored for that outcome. A chunk landing into several repositories is landed only
  when every flow it submitted to says so; a repository that landed before another refused stays landed, and the chunk
  reconciles as it does today. Each landing is recorded as an event naming the chunk, its project, the branch, and the
  revision the branch now carries.
- **The line is kept from what is true, not from what was announced.** Every few seconds the hub compares the chunks
  standing at a hub step with the submissions in line: a chunk standing there with no place is given one, and a place
  whose chunk has moved on is closed. No path a chunk takes onto a hub step — the hub's own move from one hub step to
  the next, a requeue, a resume, an upgrade — can leave it waiting in a line it is not in, and nothing that has left a
  step can keep a line waiting behind it.
- **A chunk handed to a person keeps its place.** A chunk escalated to `needs_human` while in line is passed over, and
  the branch moves on without it. An outcome its graph has no edge for goes to a person at once rather than being tried
  again. When a person requeues the chunk, it takes its place again at the back of the line.
- **Today's landing is a batch of one.** Nothing in this slice batches, reorders, or rebases. A chunk is landed as the
  slot lands it now, only behind the work queued on its own branch rather than everything in the tenant, and a chunk
  paused before it reached its hub step now waits for its resume rather than being run while paused.
- **The waiting is visible.** The board shows each flow as a lane: what is landing now, what is queued behind it, and
  what it last landed or turned back. A chunk's card says which flow it waits on and where in the line it stands, and
  links to the record of the run that landed it.
- **An operator's levers still mean what they meant.** Pausing a chunk in `delivering` is refused, as it is today.
  Stopping one withdraws its submission from every flow still holding it; one already landing runs to its verdict first.
  Restarting a chunk onto an earlier node withdraws it the same way.
- **A branch is never frozen by a turn that cannot end.** A turn whose ending failed to record is noticed by the hub
  within seconds and ended, so a branch never waits on work nobody is doing.
- **Carried over in place.** A chunk mid-delivery when the hub restarts onto this slice resumes as a submission at the
  head of its branch's flow, and the tenant-wide slot is retired once nothing holds it.

## Left for later

The flow learns to batch, to bring work up to its base, and to prove a combination in the `trains` slice; until then
turns go first come, first served, and each landing completes when its checks do. Which policy lands a repository's work
stops being the graph's choice in the `policies` slice, and `land` runs the repository's policy from then on. An
operator's hand on a flow itself — holding a branch closed to landings through an incident or a release freeze, and
releasing it — waits until someone needs it; until then a branch is held by stopping the chunks in its line.
