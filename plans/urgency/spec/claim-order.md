# Claim order

## Two orders over one queue

The ready queue keeps the order an operator gives it, and gains a second order derived from it. **Rank order** is the
queue as it stands today: each chunk at its newest explicit position, or at its promotion instant where it was never
positioned, ties broken on the chunk's id. **Claim order** sorts the same chunks by the place of their urgency in the
tenant's set, most urgent first, and by rank order within each urgency. Nothing about claim order is stored: it is
worked out each time it is read, from the ranking facts, each chunk's urgency, and the set's position facts, read
together.

Rank order is what an operator arranges, and it stays what the board's lists and the queue page show
([surfaces.md](./surfaces.md)). Claim order is what a runner reaches in, and every read that decides or explains a claim
walks it. Reordering the queue changes rank order and so claim order within a group; changing a chunk's urgency or
reordering the set changes claim order and leaves rank order alone. The backlog has no claim order, since nothing claims
from it: a backlog chunk's urgency is only a signal to the operator choosing what to promote.

The queue operations gain claim order beside the ready order they already answer, from the same ranking read and one
read of the set's order. Every chunk in the ready queue carries a live urgency, so every one has a place.

## The peeks walk claim order

Both peeks the hub serves a runner walk claim order in place of rank order, and change nothing else:

- **The matched peek** (`POST /api/fleet/queue/peek`) walks claim order exactly as it walks the queue today: skipping a
  chunk of a project the runner does not serve, or through a binding flagged against its host, and judging every other
  chunk by its blocked marking and by whether the runner can run its current node. Under `pass_over` it hands back the
  first chunk that is neither blocked nor beyond the runner, which is the first claimable chunk of the most urgent group
  that has one. Under `hold` it stops at the first unusable chunk in claim order and hands back nothing — per project,
  when the peek names its projects, as it does today. A runner that holds has chosen strictness, and claim order does
  not soften it: an urgent chunk it cannot use holds its project's less urgent work behind it, as a chunk ranked first
  does today.
- **The legacy peek** (`GET /api/fleet/queue/peek`), kept for released runners, answers the whole queue in claim order.

An entry's `position` is its place in the order the peek walked, as it always has been: in claim order now. The claim
itself never reads rank and does not start to; it still refuses, in its fixed order, what the peek may have offered
before a race.

## Prerequisites are not borrowed

An urgent chunk standing on an unfinished prerequisite carries its blocked marking as any chunk does, and the walk
passes over it or holds at it as the runner's policy says. The prerequisite stands in its own group at its own rank and
is reached when its turn comes. No read raises it, and no write is made on its behalf. The board shows what a chunk
stands on, and the next-claim explanation names the prerequisite that is holding an urgent chunk back, so the operator
who wants it sooner raises the prerequisite's urgency or its rank themselves.

## Runners of every release

Which chunk a runner reaches for was always the hub's decision: the matched peek hands back one entry, and the runner's
own pick over a peek it holds keeps the order the hub gave it. So a runner of any supported release, peeking a hub that
carries this epic, reaches for the urgent fix first without knowing urgency exists, and a released runner on the legacy
peek, which reads the whole list front to back, does the same. The peek gains nothing a runner must read, so no runner
needs upgrading.

## The walk, explained

The walk the matched peek takes becomes a pure function of the queue model that returns, beside the entry it chose, the
trace of how it got there: each chunk it passed in claim order before the one it chose, with why. The peek uses only the
choice; the explanation below uses the trace. Because both are the one function, the explanation says what the peek
would do given what the hub holds of the runner, and a unit test holds them together on every fixture the peek's own
tests use. What the runner sends with a real peek — the capabilities it has at that moment, the projects it has room for
— can still differ, so the explanation is the hub's best account, not a promise.

A passed chunk's reason is one of:

| Reason       | When                                                                                            |
| ------------ | ----------------------------------------------------------------------------------------------- |
| `blocked`    | it carries a blocked marking; the trace names the prerequisite the marking names, and the count |
| `ineligible` | the runner cannot run its current node, by the runner's capability snapshot                     |

A chunk of a project the runner does not serve, or serves only through a binding flagged against its host, is never
walked, so the explanation counts such chunks rather than listing them. `reason` is an open vocabulary
(`bzh:open-response-vocabulary`).

`GET /api/runners/{runner_id}/next-claim` answers one runner's explanation, under the permission the runner views
already ask, `fleet:view`. The hub walks claim order on the runner's behalf, from what it holds of that runner — its
served projects and bindings as the hub holds them, and its capability snapshot from its latest registration — and
answers:

- `next`: the chunk the runner reaches for when it next has room, with its urgency and its place in claim order, or none
  when nothing in the queue is claimable by it;
- `passed`: each chunk ahead of `next` in claim order within the runner's projects, with its urgency and its reason, at
  most twenty, and `passed_total` beside them;
- `if_holding`: what a runner that holds would reach for instead, walking per project as a real runner's peek does, and
  where each of its projects stops — so the board can say both, since the policy is the runner's own configuration and
  the hub holds no record of it;
- `unserved`: how many chunks ahead of `next` it never walked.

The explanation reads nothing a peek does not read, once per request, and its trace walks a list the read has already
materialized, so it costs what one peek costs (`bzh:live-set-read`). A runner that is retired, paused, or holds every
agent it runs still has an explanation: it says what the runner would reach for, and the runner's own view says why it
is not reaching now.
