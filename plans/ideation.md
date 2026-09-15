---
epic: ideation
refinement: refined
slices:
  - name: full
    status: in-progress
---

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

So it lands in the same tab, as routines beside the others, producing proposals that wait on the same two verbs. A
routine already points at whichever graph it names, and already offers only the scopes it has been given, so three new
routines need nothing from the routine model itself.

## What the platform is missing

The garden platform was built for a pass that judges code against a written rule, and in four places that assumption
reaches past the rule into the machinery. Each is small, and none of them is ideation-specific.

**A proposal need not cite a finding.** Today the hub refuses a proposal with no findings behind it, on the grounds that
an opinion with no evidence is preference dressed as a result. But an ideation pass has no observation to cite. "The
board cannot tell you why a chunk is stuck" is not evidence for the idea — it *is* the idea, stated as a lack, and
recording it twice to satisfy a check files one sentence and calls the second one justified.

The better home for that discipline is the graph's own prose, not a validator. The gardening graph's prompts tell its
proposals to answer findings, and they keep doing so; the ideation graph's prompts do not. The hub stops counting. That
is the whole change: a proposal's finding links already live in their own table and already default to none, and
`introduced`, the per-repository revisions, and the run's measurement are all optional today, so a run that cites no
finding, no commit, and no revision delivers the moment the minimum is gone.

**A run can read what was declined.** A passed proposal carries the reason it was passed, and that reason is what keeps
an idea rejected in March from arriving again in April in different words. The operator can read it; a run cannot. The
fleet's own read of a routine's proposals serves only the open ones, closures stripped. It gains the closed ones,
reasons and all, still scoped to the leased chunk's own routine.

**A run can read the fleet's numbers.** Analytics already counts which skills fire, which files are opened, and which
agent types are spawned, and spend is summed per node and per graph — but every one of those routes refuses a runner, so
the routines that most need the numbers cannot reach them. They gain a lease-scoped read of the same summaries, served
only to a chunk that is a routine run, the same way that chunk already reads its findings. Handing the worker an
operator's credentials would be smaller, and would also hand it every transcript in the fleet.

**The record can be counted, and a routine can be retired.** The epic's acceptance test is a count — proposals created,
accepted, and passed, per routine and per class — and nothing reports it yet; the routine page draws a trend of findings
and nothing of proposals. And the promise that a routine not worth waking up to costs one command to retire needs the
command, which routines do not have. Scopes already retire; routines follow the same shape.

## What the graph does differently

An ideation routine cannot point at the gardening graph, and the departures are worth stating because each one is a rule
that graph holds deliberately.

**Its survey judges intent, not a standard.** `garden-routine` refuses any impression that cites no standard, and when
the routine's axis is undeclared it records that gap as its only finding. Ideation has no rule by construction. It reads
mission, vision, and personas — documents that say what the product is for, never what the code must look like — and its
survey is written to judge against intent and to say so plainly, so that nobody later mistakes it for enforcement.

It finds its charge the way gardening does: the routine's name is an axis in the target's gardening-axes registry, and
the entry says what the routine reads and where the charter lives. The graph names no path of its own, so one packaged
graph serves any deployment with a charter to read.

**Its cross-reference reads proposals, not findings.** Gardening reconciles new observations against the routine's live
findings. Ideation reconciles new ideas against what this routine has already proposed: the open ones, and more
importantly the closed. The session doing the matching enters fresh, exactly as gardening's does — the mind that spent
an hour convincing itself an idea is good should not be the one deciding it is new.

**Nothing is scoped to a revision.** An ideation run reads the product as it stands, so it runs full every time, records
no revision, and cites no commit.

## The routines blizzard declares

Three, each named for what it reads, each declared as an axis in `blizzard-context`'s gardening registry beside the axes
the garden already tends.

**`ideation:usage`** reads the fleet's own numbers — which skills fire, which agents are spawned, which context files
are ever opened — against what the fleet was built to do. It is the one running on evidence rather than taste, and the
reason to want it most. `epic:transcripts` and `epic:analytics` both landed, and the single question anyone has put to
them so far overturned an assumption: agents constantly, skills nearly never. That was an ideation finding that arrived
by accident. This routine is the apparatus for finding the rest on purpose — and it waits on the read of those numbers
above, so it is not the one to run first.

**`ideation:features`** reads one surface at a time — the board, the CLI, the hub API — against the charter, and asks
what the people described in `charter/personas.md` still cannot do. It needs nothing but the code and the charter, which
makes it the first routine to run: it proves the whole loop, from survey to a triaged proposal, before either data read
exists.

**`ideation:cost`** reads what the fleet spends against what it produces. The core slice of `epic:cost` landed with
usage facts, budget caps, and model routing, so the numbers are all there; what nobody has done is ask them what to
change.

One is deliberately absent. Architecture belongs to the garden, which already tends it by judging drift from the
constraints `blizzard-context:/architecture/` declares. An ideation routine over the same ground would propose changing
those constraints — the same code read in the opposite direction — and two routines arguing about one codebase is a
confusion no routine name resolves. If the gardening axis leaves a real gap, that is the evidence for adding this one
later.

Their scopes — the charter, a named surface, the cost profile — join the deployment's scope list, and each routine is
offered only its own.

## The vocabulary

Three classes, declared by the ideation graph and stored by the platform without ever being read.

**`direction`** is a capability worth exploring, larger than any single work item. The operator accepts one without
minting a work item — a choice accept already offers — and the accepted idea earns a row in `epics.md`, written by hand.
This is the class the epic's promise rests on — the registry stays the operator's, and the fleet earns a voice in it
rather than a key to it.

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

**Not a scheduler.** Manual kickoff is the trigger, exactly as gardening's is. The cadence a routine keeps is
`epic:cadence`'s to build, shared with the garden rather than built twice here.

**Not a briefing document.** The capability was first framed as a short brief of feature directions. A list of proposals
that can be accepted, declined, counted, and cross-referenced by the next run is worth more than a document that can
only be read, and the epic's row is rewritten to say so.

**Not authority over the registry.** Nothing here files an epic row, and the `direction` class exists precisely so that
agreeing with an idea and commissioning it stay separate acts.

**Not a class-aware hub.** Whether accepting mints work stays the operator's choice at the moment of accepting. The hub
learns no class's meaning, and a deployment's vocabulary remains its own.

**Not the transcripts.** The run reads the summaries analytics already computes, never a session's transcript.

## How it is proven

The epic entered as a deliberate maybe, and it should decide itself on evidence rather than on how the first run reads.
Once proposals can be counted, the store answers the question: proposals created, accepted, and passed, per routine and
per class. A routine whose proposals are passed month after month has told the operator plainly that it is not worth
waking up to, and retiring it costs one command. That count is the entire acceptance test, and it exists the day the
first run delivers.
