# The gardening tab

Everything in [machinery.md](./machinery.md) happens without a person in it. A run is minted, sweeps its scope, delivers
its findings and its proposals, and ends — and nobody was needed anywhere along the way. That is the design working as
intended, and it is also what makes the surface described here load-bearing rather than decorative. A discipline that
never interrupts anyone only stays a discipline if there is somewhere its accumulated judgment can be read, weighed, and
answered on the reader's own schedule.

So the tab is not a viewer over the machinery. It is the place where the human half of the loop actually happens: where
a routine is declared and set going, where a run's observations are believed or struck down, and where an opinion the
fleet formed becomes work or does not. Three mockups carry the whole of it, and the requirements below are stated
against them.

| Sheet                                                                      | The surface it settles                                                                             |
| -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| [artifacts/gardening-routines.html](./artifacts/gardening-routines.html)   | What a deployment has declared, each routine's standing health, and the act of running one by hand |
| [artifacts/gardening-findings.html](./artifacts/gardening-findings.html)   | Every run this fleet has made, the findings it delivered, and the triage of them                   |
| [artifacts/gardening-proposals.html](./artifacts/gardening-proposals.html) | The docket of responses waiting on a decision, and the two verbs that close one out                |

They are one interactive set — the chrome is shared, the sub-tabs link across, and each sheet carries its own design
notes behind the button in its strip.

## Where it lives

Gardening is a top-level tab beside the board, not a panel inside it. The board is a surface an operator watches;
gardening is one they visit, and the two rhythms do not share a screen well. The app shell is untouched: same header,
same nav strip, one more entry.

Inside it, the tab divides along the machinery's own nouns — scopes, routines, runs, findings, proposals — because any
other division would be one this fleet invented rather than one the domain has. Only proposals carry an urgent count in
the strip, since only proposals are waiting on somebody.

The first rollout gave runs and findings one shared sub-tab and scopes none, on the reasoning that a scope is a
vocabulary the other nouns are read through rather than a thing anybody visits. Building it proved both halves wrong.
Runs and findings turned out to be read on different occasions — a run is read once, when it lands; a finding is
returned to until it is gone — and sharing a surface made each one's filters fight the other's. Scopes, meanwhile, are
visited: retiring one, or reading which routines default to it, is a real errand with nowhere else to live. Five
siblings, one per noun, is the shape that survived contact.

## Declaring and running a routine

A routine's record is small on purpose, and the surface should not make it look bigger than it is. A name, the graph a
run executes, a default scope, and model and effort defaults are the whole of what the hub stores, and the panel shows
exactly that, with the strategy rendered beside it as read-only prose so a reader can see what this routine judges
without ever being invited to believe the hub owns it.

What the panel adds is the reading no single record could give: the routine's health. Inflow against outflow over recent
weeks, the measurement its strategy asks every run to record, and — the fact that turns rotating scoped runs from a plan
into a practice — when each of its scopes was last swept, and against which revision. A routine with one scope nobody
has ever visited is not converged, however good its trendline looks, and the surface has to be able to say so.

That table is per routine and scope both, never per scope alone. Scopes are global, so a scope another routine swept
this morning is unswept ground for this one, and a panel that read the scope's own row would report the wrong thing
confidently.

Running one is a dialog over the three options a run takes, and no more. Two of them deserve care.

**Scope** is a picker over the scopes the hub holds — every one of them, since scopes are global and not a routine's own
(`machinery.md` §Scopes are entities). Each is shown as its slug and the sentence that says what it covers, because a
list of bare slugs is a list a reader has to already know. Ordering does the work filtering would: the scopes this
routine has swept before come first, and the rest follow.

Naming a scope that does not exist mints it, and the surface treats that as the deliberate act it is rather than a
typo's silent consequence. Creating one asks for its description in the same breath — a scope with no sentence is the
one a later reader cannot tell from its neighbour — and shows near-matches on the existing list before it commits, since
a slug resembling one already there opens a second bucket that no run reconciles.

**Mode** is where the platform earns its keep, and the dialog should show that rather than assert it. A delta run names
the revision the hub already recorded for this routine over this scope, and how much has landed since — a real baseline
handed to the pass, not an instruction to work one out. Choose a scope this routine has never swept and delta has
nothing to subtract from; the surface says so and steers to a full run rather than letting a run mislabel itself.

Scopes are managed from the same panel, because the routines surface is the only place anybody has a reason to look at
one. A scope's description is editable — the sentence is what makes the picker readable, and a first guess written in a
run dialog deserves a second pass. Retiring one takes it out of every picker and changes nothing else: its findings stay
live, queryable, and attributable, exactly as a retired graph's chunks keep running. Re-enabling restores the same
entity, so a scope retired early and wanted back resumes its own bucket rather than starting a second one beside it, and
the surface says which retired scopes exist rather than making somebody remember.

Kicking a run off mints a work item, ingests it, and promotes it in one act, and the surface then gets out of the way.
It confirms the chunk and offers a link to the board. It does not rebuild the board inside the tab. A run is a chunk
like any other, and the moment gardening starts re-rendering chunks it has started duplicating a surface that already
exists and will drift from it.

## Reading what a run saw

Every run owns a row, including the ones that found nothing and the ones that refused to start. Every row states its
outcome in words, and what the run's sets summed to; an escalated run reads as escalated without relying on a colour to
say so. Between them the rows are the trendline, itemized, and a surface that hid either would flatter the garden
exactly where the truth is most useful.

The measurement and the revisions a run read live one click away, in the delta beside the list, rather than on the row
itself — the first rollout put them on the row, and thirty rows each carrying a paragraph of measurement made the list
unscannable for the one job a list has.

Drilling into a run shows what it delivered as a delta rather than as a state: what was added, what was seen again, what
the run looked for and could not find. The distinction is worth spelling out on the surface itself, because it is the
one that makes a scoped run honest — a finding the run never visited carries no entry and keeps its last word, and
silence about it is never a claim about it.

From there the reader is looking at findings, and findings are where the tab stops being a report and starts being a
tool.

## Triage, and the states a finding can reach

A finding is one instance — one thing somebody could fix — and the surface has to honor that granularity without making
it unusable. Every state change carries a note: a finding struck without a reason teaches the routine nothing, and the
next run raises its twin.

Triage is dispatched one finding at a time, from the selected finding's own panel. The first rollout planned a
multi-select bulk action for the seventeen-instances-in-a-package case, and the note was to stay visible on the row
afterward; neither survived. A bulk strike wants one note covering every row it touches, which is the reason least
likely to be true of all of them — the note that teaches the routine anything is the one written against a finding
somebody actually read. The row lost its note for the same reason the list lost its checkboxes: the bucket is a place to
find the finding you want, and the panel beside it is where the finding is read and answered. `supersede` stays a CLI
and API verb, since it needs the absorbing finding's id and the panel collects no second id.

The exits themselves need a decision this plan has not yet made. [machinery.md](./machinery.md) names three ways a
finding leaves the live set: the work behind it lands, a person withdraws it, or a person accepts a run's report that it
no longer reproduces. **The mockups propose splitting that middle exit into three**, and the case is worth weighing
here.

Withdrawal today covers two facts that point in opposite directions. *We looked and decided to keep it* says something
about the code. *The pass was wrong* says something about the strategy — and a routine whose findings are struck as
mistaken at any real rate is one whose prompts need sharpening, or whose judgment does not belong at the model rung at
all. Collapsed into one verb, both disappear into a number nobody can read. Split into **won't fix**, **not a finding**,
and **superseded**, the second becomes the routine's own precision, shown beside its findings, and the first becomes an
argument for a carve-out in the standard the pass is judging against.

The split pays for itself in the trend, too. Only a resolution counts as outflow, because only a resolution tended
anything; a week of energetic triage that struck twenty findings as mistaken should not read as a week of gardening.
Whichever way the exits are finally cut, that separation is the requirement.

The gone flag stays a flag throughout. A run reporting that a finding no longer reproduces surfaces it, tinted, with the
run's own words attached, and leaves it live until a person agrees. Nothing else in the tab is allowed to close a
finding on a pass's word.

## The docket

Proposals are the one place gardening genuinely waits, and they are read in batches across runs rather than a run at a
time. That batching is what lets a run go end to end with nobody in it, so the docket is filtered by what a person
actually triages along: whether it is waiting, and what kind of response it is.

Reading one means reading the case and then the evidence. The case is prose written to be decided on without re-opening
the findings; the evidence is the findings themselves, linked and live, the same rows the findings surface triages.
Linked rather than pasted, so the inventory stays queryable while the proposal stays readable.

Two verbs close a proposal out, and both leave a record. **Passing** is not a dismissal — it is the note that stops a
later run raising the same response as though it were new, and over a lineage it is what makes "does anyone ever want
this kind of response?" a query rather than somebody's memory. Passing wants a reason more than accepting does, and the
surface should ask for one.

**Accepting** records agreement, and the surface has to hold the fact that agreement is not always a commission. Minting
a work item is the default and stays one click away, carrying the proposal's body unless the person supplies a different
one. Declining to mint is available and deliberately shaped as the more effortful path, because the failure modes are
not symmetric: a spurious backlog item is visible and deletable, while a real commission that silently created nothing
is a decision nobody can find again. An acceptance that mints nothing says so on the record, rather than showing an
accepted proposal with an empty space where its work should be.

Two things acceptance does not do, and the surface should make both legible. It does not promote — the item lands in a
rankable backlog behind the gate that already exists, because a fleet's opinion becoming work by fiat is the thing this
whole design refuses. And it does not move the findings. Work being under way is not an observation that the ground
changed, so a finding whose proposal was accepted shows its work item beside it and stays open until a run reports it
gone or a person withdraws it.

## What the tab deliberately does not do

It does not author strategy. What a pass judges by lives in the deployment's own corpus, and a text box here inviting
someone to type a standard would create a second copy of a fact the project already owns.

It does not rebuild the board, the graph editor, or the backlog. Every one of those exists, and gardening links to them.

It does not schedule anything. No cadence field, no next-run time, no pause switch — manual kickoff is the only trigger
this epic builds, and the strip says so rather than leaving a reader to wonder where the setting went.

And it does not do anything a CLI verb cannot. Every act on these sheets has one behind it, because the operator working
from a terminal and the operator working from a browser should be doing the same things to the same entities.

Printing that verb on the surface was the first rollout's way of showing it, and it half-survived. Where the verb is a
property of the thing being read — a proposal's own pass and accept — it sits on the panel beside the proposal it names
and earns the room. Where it is a property of a form being filled, it did not: a run's verb changes under the operator's
hands as they pick a scope and a mode, and a line restating the form somebody is already looking at is a second copy of
that form rather than a second way into it. Those dialogs print nothing now. The guarantee is parity — every act has a
verb, and the verb is documented where verbs are documented — not that every sheet recites one.

## Open questions the sheets raise

**What a pass sees of triage.** A withdrawn finding leaves the live set, which means the verb a running pass calls stops
returning it — and the next run is free to raise its twin. Whether withdrawn findings are exposed to a pass as
already-judged is the difference between triage that holds and triage that is redone every month.

**The campaign shape.** A remediation proposal over forty findings is one work item, not forty. Whether accepting one
may instead fan out cheap per-finding chunks, and how their outflow attributes back to the findings, is a question the
accept dialog gestures at and does not answer.

**Superseding a proposal.** Two runs weeks apart can reach the same response. Cross-referencing is meant to catch it;
when it does not, the only move the docket offers is passing the older one with a note, which works and reads badly.
