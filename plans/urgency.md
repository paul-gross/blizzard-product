---
epic: urgency
refinement: refined
status: horizon
---

# Plan — `epic:urgency`

The ready queue has one order, and the operator sets it by hand the way a team grooms a Jira backlog: drag a card up and
the fleet reaches it sooner. That order answers "what next?" well while everything in it is the same kind of work. It
answers badly the night a production bug arrives behind forty feature chunks. The operator can drag the fix to the top,
if they are awake to do it. Meanwhile the routine sweep, the gardening pass, and the small jobs nobody is waiting on sit
in the same line as the features, competing for runners they only ever needed when there was nothing better to do. What
the queue cannot hear is that some work should jump the line the moment it exists, and some should only ever take a
runner nobody else wants.

Urgency is that second axis. Every chunk carries an urgency drawn from a short, ordered set the operator keeps, and a
runner reaching for work looks at the most urgent group first, takes the highest-ranked chunk it can claim there, and
steps down to the next group only when the one above has nothing to offer. Rank keeps its meaning: it still orders the
queue, and still orders the work within each urgency. A critical fix marked urgent at the bottom of the queue is the
next thing claimed without anyone dragging it anywhere, and a small job in the lowest group waits until there is truly
nothing else. A small job that never runs under a full queue is the epic keeping its promise, so nothing ages a chunk
from one group into the next.

The word is chosen so the two axes never blur. Rank is where a chunk stands; urgency is how soon a runner should reach
for it. *Priority* would have meant both, which is why it is left unused here.

## What to build

- **An ordered set of urgencies, one per tenant.** Each urgency is held the way a routine is: an identity that never
  changes, a slug fixed when the urgency is created and unique in its tenant, retired urgencies included, and a display
  name the operator can change freely, which need not be unique and takes the slug's value when none is given. The
  operator creates the groups, orders them, and gives each a color and, if they like, an emoji or an icon of their own,
  while the hub runs, without a redeploy, from the board, the CLI, and the API alike. The set belongs to the tenant, so
  every project in it speaks the same urgencies and an urgent chunk means the same thing on every board. A new tenant
  starts with a single urgency, `standard`, so there is something to work with on the first day; everything beyond it —
  an `urgent` above, a `background` below — is the operator's to add, and nothing obliges them to keep `standard`
  either.
- **A default the operator chooses.** A tenant setting names the urgency every new chunk takes when nothing says
  otherwise. A new tenant points it at `standard`, and the operator can point it at any urgency in the set. The urgency
  it names cannot be retired while it is the default: an operator who wants it gone points the default elsewhere first.
- **Every open chunk carries a live urgency.** A chunk holds its urgency by identity, so changing an urgency's display
  name or moving it up or down the set changes nothing about which chunks carry it — they simply move with it. There is
  no empty urgency to fall into, and no surface can set one.
- **Retiring an urgency asks where its work goes.** A retired urgency leaves every choice — no chunk, routine, or chore
  can be given it again. Retiring one that open work still carries asks the operator where that work goes instead,
  offering the tenant's default, and every open chunk that carried it moves there in the same act, the way retiring a
  priority in a ticket tracker asks which priority its tickets should take. The operator decides at the moment of
  retiring, with the work in front of them, so retiring `urgent` never quietly sends five urgent fixes below the
  housekeeping. Finished work keeps the urgency it ran under, retired or not, because that is part of its history: how
  quickly urgent fixes landed last quarter is still a question worth asking after `urgent` is gone. A retired urgency
  can be revived under its own slug, back in the set and choosable again, and because no other urgency can ever take
  that slug, the finished work that carried it still means what it did.
- **What names an urgency in advance falls back, never falls through.** A routine's default and a chore's override each
  name a live urgency or name none, and naming none takes the default one level up. Retiring an urgency one of them
  named leaves it naming none.
- **Declared like the rest of the tenant's configuration.** The configuration document a tenant applies can declare its
  urgencies and its default, and a routine's default and a chore's override within the routines and chores it already
  declares. A document that names a retired urgency is refused rather than taken as a revival: retiring and reviving are
  acts an operator takes on purpose, never a side effect of applying a file.
- **Every tenant has urgencies on the day this lands.** A tenant that exists before urgency does is given the same
  `standard` urgency a new tenant starts with, set as its default, and every chunk it already holds carries `standard`,
  so the queue claims exactly as it did the day before until the operator adds a second urgency.
- **Urgency on every chunk, set the way rank is.** Every surface that edits a chunk sets and changes its urgency — the
  board, the CLI, and the API — at mint or at any time after, in the backlog and the ready queue alike, without touching
  anything else about the chunk. In the backlog it changes nothing about what the fleet does; it is a signal to the
  operator deciding what to promote, and it travels with the chunk into the ready queue.
- **Work that arrives already urgent.** Work should not depend on someone being awake to say how urgent it is. A routine
  can carry a default urgency, so an operator who keeps a `background` group can have every gardening pass enter there,
  and a nightly security sweep enter as urgent, without anyone saying so each time; a routine that names none takes the
  tenant's default. Running a routine takes an urgency of its own as an override, and a run started without one takes
  the routine's default. A cadence runs routines the same way: each of its chores either takes the routine's default or
  names an override, so one routine can run in a low group on its weekly chore and as urgent on the chore an operator
  set up for the week before a release. Cadence decides when a routine's work is minted; urgency decides how hard it
  competes once it is there, and this epic adds the override to the chore as it adds it to a run started by hand.
  Whatever work arrives with, the operator can change it afterwards like any other.
- **The queue looks the same and says more.** Both lists keep their rank order on the board, and a chunk whose urgency
  is out of the ordinary says so on its card, so the urgent fix at the bottom of the queue is the first thing an eye
  lands on. [How urgency appears](#how-urgency-appears) owns what each surface shows.
- **Urgency first, rank within it, at the moment of claiming.** The hub hands a runner the queue in claim order — most
  urgent group first, rank within each group — across every project the runner serves, and the runner takes the first
  chunk it can claim, as it does today. Because the order lives in what the hub serves rather than in how a runner reads
  it, a runner built before urgency existed reaches for the urgent fix first without knowing why. Everything a claim
  already skips — a chunk standing on an unfinished prerequisite, a chunk a runner's tags refuse — is skipped the same
  way, and the runner takes the next claimable chunk in the same group before stepping down to the next.
- **Urgency is the chunk's own, and a prerequisite does not borrow it.** An urgent chunk standing on an unfinished, less
  urgent chunk is skipped like any other blocked chunk, and the chunk beneath it waits its turn in its own group. The
  board already shows what a chunk stands on, so an operator who needs the urgent work sooner says so directly: they
  raise the prerequisite's urgency or rank it higher. The fleet never quietly promotes work on their behalf.
- **The next claim, explained.** Each runner has its own next claim, since each serves its own projects, and the board
  shows, for each runner, which chunk it will reach for next and why, so an operator who sees an urgent chunk passed
  over can tell at once what is holding it back.

## How urgency appears

An operator keeps the urgency set where they already keep the rest of what a tenant is made of, beside the work sources
and repositories on the same administration screen. Nothing about editing an urgency should feel new: it is listed,
created, edited, reordered, and retired the way those other entities are, through the same forms and the same gestures,
so an operator who has added a repository already knows how to add an urgency. The tenant's default for new chunks is
set in the same place.

On the board, the ordinary stays quiet. A chunk carrying the tenant's default urgency looks exactly as a card looks
today, because most work is ordinary and a board where every card wears a badge is a board where no badge is seen. A
chunk carrying any other urgency wears a pill at the very top of its card, filled with that urgency's color and showing
its emoji or icon, or its name where it has neither. Which urgency counts as ordinary is the tenant's own setting, not a
fixed name: an operator who makes `normal` their default sees `normal` cards plain and everything else marked. The pill
appears wherever the urgency can still change what happens next — the backlog, the ready queue, chunks waiting on or
needing a human, and running work — and is left off finished work, where the question of how soon a runner should reach
for it has already been answered.

The list view carries the same pill just to the right of the chunk's identifier, in the same color, under the same rule:
shown when the urgency is anything but the default, and never on finished work.

The CLI says the same thing in text. Its chunk and queue listings show each chunk's urgency by slug, and list in rank
order as the board does, so what an operator reads in a terminal matches what they see on the board.

## How urgency meets tagging

Urgency is built on its own, and `epic:tagging` reads it rather than holding it. Each chunk presents an inferred tag,
`urgency:<slug>`, derived from the slug of the urgency it carries and never set by hand, so a runner that declares it
takes only `urgency:background` work does so through the tags it already speaks. Changing a chunk's urgency changes the
tag with it.
