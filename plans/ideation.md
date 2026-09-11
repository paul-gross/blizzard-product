# Plan — `epic:ideation`

The operator decides what blizzard becomes, and they decide it largely from memory. They know the product they meant to
build, they remember most of what it does, and the distance between those two only shows up when something forces them
to look. Meanwhile a fleet reads this codebase every night — the surfaces, the work moving through them, the numbers it
throws off on the way — and has no way to say anything about any of it that is not a bug or a cleanup. It is the
best-read participant in the project and the only one nobody asks what the product should become.

This epic asks it. A pass reads the software as built against the charter that was meant to govern it, and hands back a
short list of directions worth exploring. Not work and not a plan — opinions of the kind a colleague offers across a
desk, which the operator weighs and mostly declines.

## It is gardening

The instinct is that this wants machinery of its own, and it does not. `epic:garden` already built a named, repeatable
pass that reads ground, forms an opinion, and puts it in front of a person, and nothing in that platform says anything
about pruning: a routine is "a named, repeatable evaluation pass", a run delivers "what it observed" and "what it
believes should be done about them". The weeding lives entirely in what blizzard brings to it.

Which is right, because a garden is grown as much as it is pruned, and blizzard's own already leans that way. The
`verification` axis hunts what is missing rather than what is excess. Two of the four proposal classes — `prevent` and
`mechanize` — author something new rather than remove anything. Ideation is not a sibling discipline to gardening. It is
more of it, pointed at what the product could be instead of at what it has accumulated.

So it lands in the same tab, as routines beside the others, producing proposals that wait on the same two verbs.

## The one thing the platform is missing

A gardening pass observes ground and then says what to do about it, and the platform requires the second to name the
first: a proposal carries at least one finding id, because a proposal with nothing behind it is an opinion the run was
not asked for. That rule is right where a pass judges code against a written standard. The observation is the evidence,
and a response with no evidence is preference dressed as a result.

An ideation pass has no such observation. "The board cannot tell you why a chunk is stuck" is not evidence for the idea
— it *is* the idea, stated as a lack. Recording it as a finding and then again as a proposal files one sentence twice
and calls the second one justified.

So the requirement becomes the routine's own: a proposal cites findings unless its routine is one whose strategy
produces none. The wire shape drops its minimum and the delivery script asks the routine instead of asking every
proposal. What kept the original rule honest survives intact, because the charge is what asked for the opinion — a
routine declared to read the charter and propose directions asked for exactly this, and the platform can tell because
the routine says so.

This is cheap. A proposal's finding links already live in their own table and already default to none, so the change is
validation and nothing else. Nor does anything need loosening on the git side: `introduced`, the per-repository
revisions, and the run's measurement are all optional today, so a run that cites no commit and names no revision already
delivers.

## What the graph does differently

An ideation routine cannot point at the gardening graph, and the departures are worth stating because each one is a rule
that graph holds deliberately.

**Its survey judges intent, not a standard.** `garden-routine` stops and escalates when the charge names a standard that
does not exist, on the grounds that a routine judging by an unwritten rule is judging by its own taste. Ideation has no
rule by construction. It reads mission, vision, and personas — documents that say what the product is for, never what
the code must look like — and its survey is written to judge against intent and to say so plainly, so that nobody later
mistakes it for enforcement.

**Its cross-reference reads proposals, not findings.** Gardening reconciles new observations against the routine's live
findings. Ideation reconciles new ideas against what this routine has already proposed: the open ones, and more
importantly the closed. A passed proposal carries the reason it was declined, and that reason is what keeps an idea
rejected in March from arriving again in April in different words. The session doing the matching enters fresh, exactly
as gardening's does — the mind that spent an hour convincing itself an idea is good should not be the one deciding it is
new.

**Nothing is scoped to a revision.** An ideation run reads the product as it stands, so it runs full every time, records
no revision, and cites no commit.

## The routines blizzard declares

Three, each named for what it reads.

**`ideation:usage`** reads the fleet's own numbers — which skills fire, which agents are spawned, which context files
are ever opened — against what the fleet was built to do. It is the one running on evidence rather than taste, and the
reason to build it first. `epic:transcripts` and `epic:analytics` both landed, and the single question anyone has put to
them so far overturned an assumption: agents constantly, skills nearly never. That was an ideation finding that arrived
by accident. This routine is the apparatus for finding the rest on purpose.

**`ideation:features`** reads one surface at a time — the board, the CLI, the hub API — against the charter, and asks
what the people described in `charter/personas.md` still cannot do.

**`ideation:cost`** reads what the fleet spends against what it produces. The core slice of `epic:cost` landed with
usage facts, budget caps, and model routing, so the numbers are all there; what nobody has done is ask them what to
change.

One is deliberately absent. Architecture belongs to the garden, which already tends it by judging drift from the
constraints `blizzard-context:/architecture/` declares. An ideation routine over the same ground would propose changing
those constraints — the same code read in the opposite direction — and two routines arguing about one codebase is a
confusion no routine name resolves. If the gardening axis leaves a real gap, that is the evidence for adding this one
later.

Their scopes — the charter, a named surface, the cost profile — join the deployment's single global scope list, which
makes a refinement the machinery already anticipated come due sooner: whether a routine should be offered only the
scopes it tends.

## The vocabulary

Three classes, and the platform stores them without ever reading them.

**`direction`** is a capability worth exploring, larger than any single work item. Accepting one mints nothing; it earns
a row in `epics.md`, written by hand. This is the class the epic's promise rests on — the registry stays the operator's,
and the fleet earns a voice in it rather than a key to it.

**`tweak`** is a concrete change small enough to be work. Accepting mints an item carrying the proposal's own body.

**`retire`** is something built that nobody uses. It is the only class that subtracts, and the only one the usage
numbers can prove outright rather than argue.

## The triage is the machinery

Every run's output is read and closed out. The operator takes what a run proposed and either accepts it or passes it,
and passing carries its reason.

That habit is not administrative tidiness; it is what makes the whole shape work. A passed proposal is a decision the
next run reads. An accepted one leaves for `epics.md`, where the next run reading the charter finds the idea already
planned and says nothing about it. The open set stays small because it is worked rather than accumulated, and the ideas
that survive pile up where ideas ought to — in the registry, human-owned, in priority order.

## What this epic is not

**Not a scheduler.** Manual kickoff is the trigger, exactly as gardening's is. The cadence field and the sweep that
reads it are the one small addition the gardening machinery already describes as its end state, and they arrive shared
with `epic:garden` rather than built twice here.

**Not a briefing document.** The capability was first framed as a short brief of feature directions. A list of proposals
that can be accepted, declined, counted, and cross-referenced by the next run is worth more than a document that can
only be read, and the epic's row is rewritten to say so.

**Not authority over the registry.** Nothing here files an epic row, and the `direction` class exists precisely so that
agreeing with an idea and commissioning it stay separate acts.

**Not a general loosening of proposals.** Findings stay required for every routine that produces them. The relaxation is
a property of the routine's declared strategy, not a hole in the format.

## How it is proven

The epic entered as a deliberate maybe, and it should decide itself on evidence rather than on how the first run reads.
The store already answers the question: proposals created, accepted, and passed, per routine and per class. A routine
whose proposals are passed month after month has told the operator plainly that it is not worth waking up to, and
retiring it costs one command. That count is the entire acceptance test, and it exists the day the first run delivers.
