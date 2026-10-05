---
epic: urgency
refinement: scaffolded
slices:
  - name: full
    status: horizon
---

# Plan — `epic:urgency`

The ready queue has one order, and the operator sets it by hand the way a team grooms a Jira backlog: drag a card up and
the fleet reaches it sooner. That order answers "what next?" well while everything in it is the same kind of work. It
answers badly the night a production bug arrives behind forty feature chunks. The operator can drag the fix to the top,
if they are awake to do it. Meanwhile the routine sweep, the gardening pass, and the small chores nobody is waiting on
sit in the same line as the features, competing for runners they only ever needed when there was nothing better to do.
What the queue cannot hear is that some work should jump the line the moment it exists, and some should only ever take a
runner nobody else wants.

Urgency is that second axis. Every chunk carries an urgency drawn from a short, ordered set the operator configures —
`urgent`, `standard`, and `background` out of the box — and a runner reaching for work looks at the most urgent group
first, takes the highest-ranked chunk it can claim there, and steps down to the next group only when the one above has
nothing to offer. Rank keeps its meaning: it still orders the queue, and still orders the work within each urgency. A
critical fix marked urgent at the bottom of the queue is the next thing claimed without anyone dragging it anywhere, and
a background chore at the top waits until there is truly nothing else. A background chunk that never runs under a full
queue is the epic keeping its promise, so nothing ages a chunk from one group into the next.

The word is chosen so the two axes never blur. Rank is where a chunk stands; urgency is how soon a runner should reach
for it. *Priority* would have meant both, which is why it is left unused here.

## What to build

- **An ordered set of urgencies, one per tenant.** The operator names the groups, orders them, and gives each an
  indicator — an emoji, or an icon of their own — while the hub runs, without a redeploy. The set belongs to the tenant,
  so once `epic:projects` lands every project in it speaks the same urgencies and an urgent chunk means the same thing
  on every board. One group is the default every new chunk takes when nothing says otherwise: `standard`, out of the
  box.
- **Urgency on every chunk, set the way rank is.** The operator sets and changes a chunk's urgency from the board and
  the CLI, at mint or at any time after, in the backlog and the ready queue alike, without touching anything else about
  the chunk. In the backlog it changes nothing about what the fleet does; it is a signal to the operator deciding what
  to promote, and it travels with the chunk into the ready queue.
- **Work that arrives already urgent.** Work should not depend on someone being awake to say how urgent it is. A routine
  names the urgency of the work it mints, so a gardening pass enters the queue as background and a nightly security
  sweep as urgent, with `epic:cadence` deciding when each is minted and urgency deciding how hard it competes once it is
  there. A work item's own stated priority — a Jira ticket marked Highest — maps onto an urgency through a table the
  operator keeps, and an alert raised through `epic:signals` can arrive urgent. Whatever work arrives with, the operator
  can change it afterwards like any other.
- **The queue looks the same and says more.** Both lists keep their rank order on the board. Each card carries its
  urgency's indicator, so the urgent fix at the bottom of the queue is the first thing an eye lands on.
- **Urgency first, rank within it, at the moment of claiming.** Claim order is decided when a runner reaches for work,
  against the queue as it stands. Everything a claim already skips — a chunk standing on an unfinished prerequisite, a
  chunk a runner's tags refuse — is skipped the same way, and the runner takes the next claimable chunk in the same
  group before stepping down to the next.
- **Urgency is the chunk's own, and a prerequisite does not borrow it.** An urgent chunk standing on an unfinished
  background chunk is skipped like any other blocked chunk, and the background chunk waits its turn in its own group.
  The board already shows what a chunk stands on, so an operator who needs the urgent work sooner says so directly: they
  raise the prerequisite's urgency or rank it higher. The fleet never quietly promotes work on their behalf.
- **The next claim, explained.** The board shows which chunk the fleet will reach for next and why, so an operator who
  sees an urgent chunk passed over can tell at once whether it is blocked, refused, or behind a chokepoint.

## How urgency meets a chokepoint

A chokepoint is a wall across the rank order, and urgency does not climb it. Nothing ranked behind a chokepoint is
claimed until the chokepoint clears, however urgent it is. An operator who needs a fix to land before the big change
says so with rank, by moving the fix ahead of the chokepoint — a deliberate act, because putting work ahead of a
chokepoint is also making the chokepoint wait for it.

Ahead of the wall, urgency orders the work as it does anywhere else, and the chokepoint waits for every chunk ranked
ahead of it, background chores included. Those chores do not stall the road: once the urgent and standard work ahead of
the chokepoint is gone, the background work is all that is left to claim, and the fleet takes it. A chokepoint's own
urgency changes nothing, since by the time it can be claimed it is the only work in its project a runner can take.

## Open questions

- What happens to the chunks carrying an urgency the operator renames, reorders, or removes from the set.
- Whether urgency rides `epic:tagging`'s scoped tags as a scope that also carries an order, which would let a runner
  declare that it takes only background work, or stands on its own.
