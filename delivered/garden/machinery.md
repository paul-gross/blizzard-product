# The routine machinery

What blizzard builds, stated without a single fact about what anyone will tend. A deployment brings the judgment; the
platform brings a way to run that judgment repeatedly, keep what it saw, and put what it thinks in front of a person.
Everything here holds for a project blizzard knows nothing about.

Four nouns carry it. A **routine** is a named, repeatable evaluation pass. A **scope** is a named body of ground a
routine sweeps. A **run** is one execution of a routine over a scope, in a mode. A run delivers **findings** — what it
observed — and **proposals** — what it believes should be done about them. All of them are durable hub entities with
their own identity, lifecycle, and management surface, and findings and proposals are the only two whose *contents*
blizzard has an opinion about — a routine and a scope are records, where those two are shapes a graph writes to.

## Routines

A routine is a small hub-stored record: a name, the graph a run of it executes, a default scope, and model and effort
defaults — the graph and the scope held as references to entities the hub owns. That is the whole of it. It carries no
strategy, because the strategy lives in the graph it points at.

Routines are authored by the operator — in the gardening tab and through CLI verbs — at the moment a project has grown
enough to need one. A young project declares none, and that is the correct state rather than a gap: tending begins when
there is growth worth pruning.

The routine's name is the lineage. Every run carries it, and so does every finding and proposal that run delivers, which
turns "what has this pass accumulated" into a query rather than an errand. Names are stable, because a rename severs a
lineage from its own history. A deployment that organizes its tending around named subject areas declares one routine
per area and names each routine for the area it tends — that mapping is the deployment's convention, and the platform
sees only the name.

## Running one

`blizzard hub routine run <name>`, and the action beside the routine in the gardening tab, mints a fresh work item from
the routine, ingests it, and promotes it in one act. Manual kickoff is itself the deliberate human trigger, and for this
epic it is the only trigger there is.

A run takes three options, and each of them reaches the pass as prose:

- **Scope** — an existing scope or a new slug, defaulting to the routine's own. It says which ground this run covers,
  and naming one that does not exist yet mints it.
- **Mode** — `full` or `delta`. A full run judges the whole scope afresh. A delta run judges only what has changed since
  this routine last ran, which is the option that makes cost scale with change rate rather than with target size. The
  platform is what makes delta honest: it knows the revision the last run of this routine over this scope recorded, so a
  delta run is handed a real baseline rather than an instruction to work one out. A pair with no prior run has no
  baseline to subtract from, and a run against one is a full run whatever it is asked for.
- **A charge note** — free text, appended to the run's charge as a "This run" section, for the operator steering an
  experiment.

Because the consumer is a model reading prose, options are concatenation rather than a substitution grammar. Machine
state stays out of the preambles entirely: a running pass fetches its live context, and its routine's live findings,
through the API rather than having them pasted into a prompt at mint time.

## Scopes are entities

A run does not sweep everything; it sweeps a scope. A scope is not a path — it is a name the deployment has defined, and
what it covers is a sentence rather than a directory. `runner` may happen to resolve to a directory, but `test-files`
resolves to something scattered across a whole tree, and `public-api-surface` to a judgment about which symbols count.

A scope is a hub entity: a slug and a description, global to the deployment rather than owned by any one routine. It is
minted the first time a run names it, and from then on it is a row the store enumerates — which is what lets a run
dialog offer the vocabulary rather than ask an operator to remember it.

The slug is lowercase letters, digits, and hyphens, and nothing else. It is a key an agent types into a CLI call, a
value the store groups by, and a token that has to survive a round trip through a preamble unchanged — so the
description carries the prose and the slug never does. `test-files`, not `Test Files`; never a scope that has to be
quoted to be passed.

Minting on first use is what makes a near-miss recoverable rather than merely regrettable. A slug that resembles an
existing one still opens a second bucket — the hub resolves no scope and cannot know the two were meant to be one — but
that bucket is now a row somebody can see and act on, instead of a divergence that surfaces only when a trend stops
making sense. Scopes retire and re-enable the way graphs do (`blizzard-context:/domain/graphs/identity.md`): retiring
takes a scope out of the selection list and leaves every finding recorded under it live and queryable, and re-enabling
restores the same entity rather than minting a twin, so the bucket survives the round trip.

What the hub still does not do is resolve one. A scope is opaque — a label the store indexes, groups by, and hands back,
with the meaning living in its description and the adherence living in the agent that read it. Best effort is the honest
standard: a model deciding what falls inside `test-files` exercises the same judgment it exercises deciding a finding is
real, and no validation the hub could run would improve it. Holding the list is not reading it, exactly as indexing a
class is not interpreting one.

Every scope is offered to every routine. Whether a routine should see only the scopes it tends is a later refinement,
and nothing about it changes the entity.

## Routine and scope are a pair

A routine names what is being judged and a scope names which ground. Neither addresses a run's standing state alone, and
**the pair is the unit that does.**

Every finding carries both, so a routine's findings partition into buckets a later run fetches one of rather than a wall
it has to filter for itself. The revision a run recorded is a fact of the pair, so a delta run is handed the baseline
for this routine over this scope and no other. Last-swept is a fact of the pair too: a routine converges only when every
scope it tends has been visited, and a scope one routine swept this morning says nothing about what any other routine
has looked at.

That the pair carries the state is why none of it lives on the scope row. A scope is a global name and a sentence; when
it was last swept, against which revision, and what is open under it are answers to a question that names a routine as
well.

## Strategy is data

What a pass evaluates is the deployment's data, never blizzard's code — the same platform/content split the graphs
already honor: users bring their graphs, they bring their strategies. A routine points at a node graph, and the graph's
node preambles carry the strategy: what to read, what to look for, what to judge it against.

A strategy can be stated inline in a preamble, and for a one-off that is the shortest path. The better habit is to
point: name the project's own context files and let the pass read them where they live. A strategy restated in a
preamble is a second copy of a fact the project already owns, and it goes stale the moment that project's standards
move, where a pointer resolves against a corpus the people who own it keep current. Pointing is also what keeps a
routine agnostic, and that is the property that makes this machinery rather than a feature: the same shape works against
a project blizzard has never seen.

## A run is a chunk

A run is a chunk like any other, executing the graph its routine points at and emitting artifacts the way a coding graph
emits commits and text. Nothing about it is special at the chunk: it holds a work ref and resolves through it, exactly
as every chunk does (`blizzard-context:/domain/work/chunk.md` §Work refs). What it resolves to is where the difference
lives — an item minted from a routine, and so an item that knows which routine minted it, which scope this run was
given, and which mode it runs in. The orchestration layer stays a layer that has never heard of any of this.

The item carries the run's charge twice, for two readers that need different things. As **prose**, because what acts on
it is an agent reading a preamble, and a strategy is written to be read. As **values** — the routine, the scope, the
mode — because the store buckets findings by them and draws a lineage's trend from them, and neither is a thing to parse
back out of a sentence.

What differs downstream is the shape of what a run delivers: no commits, and artifacts that are lists rather than diffs.

## Findings are artifacts, of the same standing as commits

A chunk's artifact lane already carries commit pointers and assets. Findings join it as a third thing a node may
produce, and they are first class in the same sense a commit pointer is: minted at delivery, addressed by id, and
durable independent of the run that produced them.

Where they differ from every other artifact is that blizzard has an opinion about their shape. A commit pointer is
opaque payload the hub stores and hands back; a finding is a row the hub indexes, counts, buckets, and serves to a
management surface. It can only do that if it knows the shape, so the wire format below is the platform's, published
once and fetched at runtime rather than authored per graph (§Where the formats live). The hub validates against it and
reads nothing inside it.

### A candidate finding

A run's survey artifact is a list of candidates. Candidates have no ids, because identity is minted at delivery; each
carries a local `ref` stable only within its own submission, so a later node in the same run can name it.

```json
{
  "ref": "F1",
  "class": "stale-docstring",
  "locus": "src/billing/invoice.py:42",
  "summary": "Module docstring narrates the change history rather than the contract.",
  "introduced": "a1b2c3d"
}
```

**class** is the deployment's own vocabulary for a kind of weed. The hub indexes it and never interprets it: it can
count how often a class recurs — which is what any case for mechanizing a judgment rests on — without knowing what the
name means. Keep the spelling stable across runs, because two spellings of one class split its own recurrence count and
hide the case.

**locus** is where it lives — normally a repo-relative path, optionally `:line` or `::symbol`. The hub stores the string
and never resolves it, so a finding about a whole body of ground rather than a point inside it may name that ground
instead. What a locus means is the deployment's business, exactly as a class is.

**introduced** is best effort — the commit that introduced what the finding objects to, from `blame` on the locus. Omit
it rather than guess. A reformat defeats blame, and a rule that went stale because the code around it moved has no
introducing commit at all.

**A finding is one instance — one thing somebody could fix.** Not a theme, and not a tally: seventeen instances in one
package are seventeen findings, each with its own locus and its own id. That is what lets six of them resolve while
eleven stay open, what lets a campaign hand each one to its own cheap agent, and what makes outflow a real count rather
than a theme that stays open until its last instance falls. Grouping is not this artifact's job; the entity built for
"these belong together" is the proposal, which names every finding behind it.

## A run emits a delta, not a state

The artifact a run publishes is not the routine's new standing state — it is the set of changes to apply to it. The
state is what the accumulated deltas add up to, and no single run is ever responsible for restating the whole of it.

**Additions** carry a candidate through unchanged, minus its `ref`. The hub mints an id for each.

```json
{ "op": "add", "class": "stale-docstring", "locus": "…", "summary": "…" }
```

**Transformations** act on a finding already live on this routine, named by its id.

```json
{ "op": "observed", "id": "fin_01JKQ8Z3M4N5P6R7S8T9V0W1X2" }
{ "op": "gone",     "id": "fin_01JKQ8Z3M4N5P6R7S8T9V0W1X3", "note": "No longer reproduces at the recorded locus." }
```

`observed` says the finding still reproduces. It restamps when the finding was last seen and against which revision, and
carries no payload: the finding was true when it was recorded and it is true now, so there is nothing to revise. `gone`
says the run looked and could not find it. That does not close the finding; it flags it for a person, because a finding
leaves the live set on human judgment and never on a pass's word.

Every delivered list declares the scope it was swept under, the revision the run read per repository, and the
measurement the routine's strategy asks every run to record. All three are properties of the artifact rather than of any
finding inside it, and all three are recorded whether or not the delta holds a single entry — a clean run's datapoint is
its product.

### The rule the delta shape exists to enforce

**Emit transformations only for ground the run actually visited.** A finding the run did not look at gets no entry, and
so keeps its last word. Silence about a finding is never a claim about it.

That is what makes scoped and delta runs honest without asking anything of an agent's discipline. A run scoped to one
name emits transformations for the findings it actually examined, and findings in other scopes go untouched because
nothing in the artifact mentions them — not because the run remembered to carry them forward. Absolution by omission
stops being a failure mode and becomes structurally impossible.

It also leaves room for a run whose whole job is to act on findings after the fact — reconciling what a delivery
resolved, emitting nothing but transformations and not a single new observation.

## Identity is the hub's to assign

At delivery, every addition is minted as a row with its own `fin_<ULID>`, and each artifact's list becomes a
`fins_<ULID>` set pointing back at the run that delivered it. The set belongs to the artifact rather than to the run: a
run that publishes three lists delivers three sets, all naming the same run, which is what keeps a fanned-out graph's
partitions distinguishable after the fact.

An agent never invents an id, because minting one would be asserting a fact about the store's contents from outside the
store. That assignment is also what retires the fingerprint problem: a later run never recomputes whether two
observations are the same finding — it reads the routine's live findings, sees their ids, and names the one it means.
Matching becomes judgment expressed as a reference rather than a hash somebody has to keep stable. And because a ULID
carries its own creation instant, when a finding was first recorded is a fact of its id.

What a finding contains is the deployment's business. What blizzard indexes is the row around that body: the id, the
set, the routine, the scope, the class, the run, the timestamps, the counts. Indexing a label is not reading it — the
store counts how often a class recurs and groups findings by scope without ever learning what either name means. That is
enough to draw a trend and not enough to know what a weed is.

## Proposals

A finding is an observation; a proposal is a proposed response to one or more of them. They are separate entities
because their lifetimes differ: findings persist as evidence whether or not anyone ever acts, while a proposal is
waiting on a person from the moment it lands.

A proposal is the second artifact whose shape blizzard owns, and for the same reason: the hub persists it at delivery,
so the hub has to know the shape.

```json
{
  "ref": "P1",
  "class": "fix-the-source",
  "title": "Author a docstring standard covering change-history narration",
  "body": "…the case, in enough detail that a person can decide without re-reading the findings…",
  "findings": ["fin_01JKQ8Z3M4N5P6R7S8T9V0W1X2", "fin_01JKQ8Z3M4N5P6R7S8T9V0W1X3"]
}
```

**class** is the deployment's taxonomy for a kind of response, exactly as a finding's class is its taxonomy for a kind
of weed. The label above is an invented illustration, not a value blizzard knows: the platform stores it, indexes it,
groups by it, and never interprets it. A deployment declares its own vocabulary of proposal classes in the same strategy
its preambles point at, and what any of them mean is settled there.

That the hub stays ignorant of the vocabulary does not make the field decorative. It is the field that lets a routine be
asked what kind of response its accepted proposals have been — whether the same class recurs run after run, and whether
anyone ever passes on it. Those are the queries a deployment builds its own judgment on, and they need only a stable
label to answer.

**findings** are the ids this response answers. Required and non-empty: a proposal with nothing behind it is an opinion
the run was not asked for.

### The human on the loop

Proposals attach to their routine and go no further on their own. They are not chunks and they do not become work
without a person, because that judgment is not the fleet's to make — the pass has said what it thinks, and the decision
belongs to whoever reads it.

Two verbs close one out. **Passing** leaves the record that the proposal was considered and declined, so a later run's
cross-reference does not raise it again as though it were new. **Accepting** records agreement — and agreement is not
always a commission.

Most acceptances mint work: a self-sourced work item, linked to the proposal in the same act, carrying the proposal's
own body unless the accept supplies a different one. That body argument is optional, and it exists because an item's
text is often drafted outside blizzard — paste it in, but let the act that creates the item be the act that carries the
link.

Some acceptances mint nothing. A proposal whose response is that the problem is handled elsewhere — outside the fleet,
by hand, in a decision no work item would honestly capture — is still a proposal worth agreeing to, and forcing it to
produce an item would put a lie in the backlog. So an accept may decline to mint one, and the proposal is settled either
way. Minting stays the default and refusing it is the deliberate act, because the two failure modes are not symmetric: a
spurious backlog item is visible and deletable, while a real commission that silently created nothing is a decision
nobody can find again.

Where an acceptance does mint work, that link is the measurement story. Findings resolve because work lands, and knowing
which work resolved which findings is what separates a real inflow and outflow from two unrelated counts. An acceptance
that mints nothing makes no such claim, and it changes nothing about the findings behind it: they stay live, because a
decision about a proposal is never an observation about the ground. They leave the live set the way every finding does —
a later run reporting them gone, or a person withdrawing them.

## Where the formats live

Both shapes above are the platform's, and a garden graph must not carry its own copy of either. A graph's `artifacts:`
map bakes its entries in at mint and serves them from a local pin, which is exactly right for a declaration the graph
owns and wrong for one it merely obeys: the hub validates a submission against a shape that ships with the validator, so
a baked copy is a second owner of the same fact, free to drift the moment blizzard tightens the format under a mint
nobody re-cut.

So this epic adds a third artifact scope. Beside **node** scope — a node-step's own output — and **graph** scope — the
definition text a mint bakes in once — **system** scope holds artifacts blizzard itself publishes: read-only, named in a
global namespace rather than per-graph, and reachable by any worker in any chunk through the same lease-scoped verbs it
already uses, scope-qualified the way everything else is. This epic publishes two of them, the finding format and the
proposal format, and the facility generalizes to anything a running agent needs to know about how to interact with a
chunk operation.

System scope departs from graph scope on one point, deliberately. A graph-scope read is answered locally because a mint
is immutable and the hub can never hold a newer answer (`bzh:graph-scope-reads-local`). A system-scope read is not
pinned and must not be: its whole purpose is to hand a worker the shape the validator will actually judge its submission
by, and those two ship together. Reading it late is the correct behavior rather than a hazard — a format that changed
under a run changed the validator with it, and the worker that read the newer doc is the one whose delivery passes.

## Delivery

Every path through a garden graph ends at a hub-executed delivery node, and the platform ships the script that node
runs. It validates each artifact's shape and nothing else: that it parses, that required fields are present, that ids
are well-formed, that every finding id a transformation or a proposal names is live on this routine, and that any commit
cited resolves. It reads none of the prose, and it makes no judgment about whether a finding is worth having.

Passing that check, it mints a finding row per addition, a set per artifact list, applies the transformations, attaches
the surviving proposals to the routine, and records the run's revision and measurement. A rejected artifact routes back
into the graph with the failure attached and nothing written.

Delivery is the only place any of this becomes durable. Everything before it is working material, and nothing outlives a
run that never delivers — already the rule for proposals, and now the rule for findings too. That is also why every path
through a garden graph reaches delivery, including the ones that found nothing: a run's measurement and the revision it
read are its product whether or not it recorded a single finding, and a clean pass that never delivered would leave a
hole in the trendline exactly where the target was healthiest.

## Managing findings and proposals

Findings, their sets, and proposals are new durable hub entities, and the hub gives them a first-class management
surface rather than a log to scroll.

Gardening lives in its own tab in the hub webapp rather than a corner of the board. The board watches work in flight;
gardening is a standing discipline an operator sets up once and tunes over months, and the two deserve separate
surfaces. The tab is the home for the whole of that setup — the routines a deployment has declared, the graphs behind
them, each routine's live findings, and the proposals waiting on a decision. Proposals batch there and are read across
runs rather than one run at a time, which is what lets a run go end to end without a person in it.

Underneath, the same entities are reachable through their own endpoints and CLI verbs. One of those verbs matters to the
machinery itself rather than to an operator: `blizzard hub finding list` is what a running pass calls to read its bucket
— its routine's live findings, filtered to this run's scope. Without it the cross-reference step has nothing to read and
every run is a first run.

## What the store buys

Because findings are rows, a trend is a query. The findings created in a week are a routine's inflow and the ones
resolved are its outflow, and a routine whose inflow outruns its outflow week after week is losing ground in a way no
single run would show.

Where findings record the commit that introduced the code they object to, the same data cuts a second and sharper way.
Findings attributable to the last month's commits, held against findings attributable to older code, separate prevention
from remediation: a target whose recent-code findings are falling while its total holds flat has stopped the inflow and
simply not drained the backlog yet, which a bare count of open findings would report as no progress at all.

Nothing re-verifies a finding on a schedule, but cross-referencing re-observes as a byproduct. A finding inside a run's
scope is either seen again and transformed to say so, or absent and transformed to say that; findings outside the scope
keep their last word. So how recently a finding was observed is free exactly where it is meaningful and honestly stale
where it is not — one seen this morning in a weekly-swept package carries real confidence, while one last observed six
weeks ago is telling you nothing has looked at that ground since, which is its own useful fact and one no re-reading was
paid for.

Outflow keeps a human in it throughout. A finding leaves the live set when the work behind it lands, when someone
withdraws it, or when someone accepts a run's report that it no longer reproduces — never on the pass's word alone.

## What the platform deliberately does not do

**Fan-out is the graph's.** When a scope exceeds one context it is a node's own preamble that says to partition, and the
pass spawns sub-readers through whatever facility its harness gives it. A chunk travels one node at a time and what a
session does inside a node is its own business, so blizzard cannot tell a fanned-out survey from a monolithic one. What
the platform does owe fan-out is on the way out: a node may declare several lists, and delivery mints a set per list,
which is what keeps the partitions distinguishable afterward.

**Gates are the graph's, and the runner's.** Whether a human gate sits anywhere in a run is not the platform's call: a
runner imposes one on any node by name from its own config, reloaded every tick, without editing or re-minting a graph
(`blizzard-context:/domain/humans/gates.md` §How a gate arises). Nothing in the machinery needs a person mid-flight,
because findings are evidence that lands either way and proposals wait at the routine, so every decision worth a human
is available afterward and batched.

**Scale is a run option, not a feature.** A target too large for one context is covered by rotating scoped runs under
one routine — scope-aware convergence is what makes sharding honest — with fan-out inside each run, and after one full
sweep, delta mode is the steady state. The full sweep becomes a rare deliberate act: a new routine, or a changed
standard, which re-opens judgment on unchanged code.

**Nothing here judges.** The platform never learns what a weed is, never resolves a scope, never reads a class, and
never turns a proposal into work. A pass has no edit authority either: everything it touches changes only through
promoted chunks riding normal delivery.

## The end state

Autonomous recurrence — the pass that runs every seven days whether or not anyone remembers it — is one small addition
to a shape that will not change: a cadence field on the routine, and a hub sweep that runs any routine whose interval
has elapsed and whose previous run is not still live.

The convergent pattern is what makes a schedule safe to turn on. A run that reads its routine's live findings and
delivers a delta against them cannot duplicate what it already recorded, no matter how often it fires, so automation
becomes a trust decision per routine rather than new machinery. It stays a judgment rather than a threshold: no run
count and no override rate earns a routine its cadence, because the person reading the proposals is the one who knows
whether they are worth waking up to.

Until then, exploration is the mode — edit the charge, run it again, watch what it proposes — with the findings backlog
persisting across every experiment.

## Open question

Whether a routine can be triggered by a landing, rather than only by hand or by a clock — a pass reading one delivery's
diff while it is one commit old, instead of a scope of code on a cadence. It is a machinery question, a third trigger
beside the two this epic builds, and the sweep decides it: what a cadence run catches, and how much of that a human
reviewer had already caught on the way in, is the evidence for whether a per-delivery trigger earns its cost.
