# Plan — `epic:garden`

Standing tending of what a project accumulates. The name is the thesis: a garden is something growing, and growth is
exactly why maintenance is its steady state rather than an intervention — nothing tended holds still between visits. A
young project with little code and little context needs no gardening at all, and has no routines to its name; that is
the correct state, not a gap to fill. Gardening begins when there is growth worth pruning, and from then on it does not
end.

This is a hard problem that will take real exploration, and this epic is shaped for exploring it: it builds the
machinery for named, repeatable evaluation passes an operator kicks off by hand, watches, and refines — with fully
autonomous recurrence as the end state the machinery is built to grow into, not the thing built first.

What blizzard builds is that machinery and nothing more. It never learns what a weed is. The strategy — what to look
for, and what to judge it against — is the deployment's own. Blizzard's is written down beside this file:
[blizzard-garden.md](./blizzard-garden.md) is what it will tend, and [blizzard-graph.md](./blizzard-graph.md) is the
graph a routine runs to tend it, prebaked at [artifacts/garden-routine/](./artifacts/garden-routine/) so it can be
argued with before it is built.

## What already holds

`epic:self-sourced-work` built the lane a pass's opinions eventually travel — structured proposals on node completions,
materialized by the hub at delivery, filterable at a human gate before they exist — so nothing this fleet thinks becomes
work by fiat. What rides that lane here is narrower than the whole of a pass's output: a garden proposal a person has
accepted, and nothing else. A finding never becomes a work item, and a proposal becomes one only in the act of being
accepted. The backlog those items land in is rankable, the promote gate holds, and closure facts record how every one of
them ends.

Work already runs as graphs with human gates where a graph wants them, and the hub has no work scheduler of any kind —
the only recurring machinery is the reconciler sweep loop. That absence is now deliberate: this epic leaves it mostly
intact.

## Strategy is data

What a pass evaluates is the deployment's data, never blizzard's code — the same platform/content split the graphs
already honor: users bring their graphs, they bring their gardening strategies. A garden routine points at a node graph,
and the graph's node preambles carry the strategy: what to read, what to look for, what to judge it against. Two layers
own everything between them.

**The methodology** — how any convergent garden pass behaves, whatever it tends and whatever project it tends it in — is
written once in the workflow layer: inherit, converge, propose boundedly, report the trend, respect scope. It carries no
fact about any target.

**The graph and its preambles** are the deployment's data, and what goes into them is the operator's call. A strategy
can be stated inline, and for a one-off that is the shortest path. The better habit, and the one blizzard will use on
itself, is to point: name the project's own context files and let the pass read them where they live. A strategy
restated in a preamble is a second copy of a fact the project already owns, and it goes stale the moment that project's
standards move, where a pointer resolves against a corpus the people who own it keep current. Pointing is also what
keeps a routine agnostic, and that is the property that makes this machinery rather than a feature: the same shape works
against a project blizzard knows nothing about. Where a project wants a well-formed thing to be pointed at,
`canon:gardening-axes` is winter's convention for organizing gardening strategy inside a harness — blizzard neither
requires it nor reads it.

## Scope is a declared vocabulary

A run does not sweep everything; it sweeps a **scope**. A scope is not a path — it is a name the deployment has defined,
listed with what it covers in the same gardening strategy the preambles point at. `runner` may happen to resolve to a
directory, but `test-files` resolves to something scattered across the whole tree, and `public-api-surface` to a
judgment about which symbols count. The reference list is what makes such a name mean the same thing twice.

The name is a **slug**: lowercase letters, digits, and hyphens, and nothing else. It is a key an agent types into a CLI
call, a value the store groups by, and a token that has to survive a round trip through a preamble unchanged — so the
description carries the prose and the name never does. `test-files`, not `Test Files`; never a scope that has to be
quoted to be passed.

Blizzard stores the name and never resolves it. A scope is opaque to the hub exactly as an axis is — a label it indexes,
groups by, and hands back, with the meaning living in the deployment's list and the adherence living in the agent that
read it. Best effort is the honest standard: an LLM deciding what falls inside `test-files` is exercising the same
judgment it exercises deciding a finding is real, and no validation the hub could run would improve it.

That opacity is exactly what lets scope be the bucket. Every finding carries the scope it was recorded under, so an
axis's findings partition into buckets a later run fetches one of, rather than a wall it has to filter for itself.

## The gardening graph

A gardening run is a chunk like any other, running the graph its routine points at and emitting artifacts the way a
coding graph emits commits and text. Nothing about it is garden-shaped at the chunk: it holds a work ref and resolves
through it, exactly as every chunk does (`blizzard-context:/domain/work/chunk.md` §Work refs). What it resolves to is
where the difference lives — an item minted from a routine, and so an item that knows which routine minted it and which
scope this run was given. The orchestration layer stays a layer that has never heard of gardening.

That item carries the run's charge twice, for two readers that need different things. As **prose**, because what acts on
it is an agent reading a preamble, and a strategy is written to be read. As **values** — the routine, the scope —
because the store buckets findings by them and draws a lineage's trend from them, and neither is a thing to parse back
out of a sentence.

What differs downstream is the shape of what it delivers: no commits, and an artifact that is a list rather than a diff.
The stages a strategy will typically walk:

1. **Read** — the preamble sends the pass into the project's context, canon, and code, across the scope this run was
   given.
2. **Find** — it records what it sees, emitted as an artifact.
3. **Cross-reference** — it fetches what its lineage already knows, through a CLI verb listing the axis's live findings,
   and reconciles new against old: what is genuinely new, what is the same finding seen again, what has gone.
4. **Propose** — it drafts responses to what it found.
5. **Deliver** — the delivery node hands the run's artifacts to the hub, which is where any of it becomes durable.

Everything before delivery is working material. Nothing outlives a run that never delivers, which is already the rule
for proposals and stays the rule here.

## A run emits a delta, not a state

The artifact a run publishes is not the axis's new standing state — it is a set of changes to apply to it. New findings
arrive as additions; findings seen again or no longer reproducing arrive as transformations naming the findings they act
on. The axis's state is what the accumulated deltas add up to, and no single run is ever responsible for restating the
whole of it.

This is what makes scoped and delta runs honest without asking anything of an agent's discipline. A run scoped to
`runner` emits transformations for the findings it actually looked at, and findings in other scopes go untouched because
nothing in the artifact mentions them — not because the run remembered to carry them forward. Absolution by omission
stops being a failure mode and becomes structurally impossible.

It also leaves room for a stage that acts on findings after the fact: a run whose whole job is to reconcile what the
last delivery resolved, emitting nothing but transformations.

## Findings, sets, and identity

The artifact is a list; the hub is where a list becomes rows. At delivery every new finding is minted as a row with its
own `fin_<ULID>`, and each artifact's list becomes a `fins_<ULID>` set pointing back at the run that delivered it. The
set belongs to the artifact rather than to the run: a run that publishes three lists delivers three sets, all naming the
same run, which keeps a fanned-out graph's partitions distinguishable after the fact. Identity is the hub's to assign
and never the agent's to invent — an LLM minting an id would be asserting a fact about the store's contents from outside
the store.

That assignment is also what retires the fingerprint problem. A later run never has to recompute whether two
observations are the same finding: it reads the axis's live findings, sees their ids, and names the one it means.
Matching becomes the agent's judgment expressed as a reference, rather than a hash somebody has to keep stable. And
because a ULID carries its own creation instant, when a finding was first recorded is a fact of its id.

What a finding contains is the deployment's business — its locus, its class, its summary, whatever child findings roll
up beneath it. What blizzard indexes is the row around that body: the id, the set, the axis, the scope, the class, the
run, the timestamps, the counts. Indexing a label is not reading it — the store counts how often a class recurs and
groups findings by scope without ever learning what either name means. That is enough to draw a trend and not enough to
know what a weed is.

A finding is one instance, not a theme. One thing somebody could fix, at one locus, with one id — because an id per
instance is what lets nine hundred of a thousand resolve while a hundred stay open, what lets a campaign hand each one
to its own cheap agent, and what keeps outflow an honest count instead of a theme that reports nothing until its last
instance falls. Grouping is a job the store already has an entity for: a proposal names every finding behind it, and
that is where "these belong together" is said. What keeps a person's reading short is the query, not the shape of the
row.

## Proposals, and the human on the loop

A finding is an observation; a proposal is a proposed response to one or more of them. They are separate because their
lifetimes differ: findings persist as evidence whether or not anyone ever acts, while a proposal is waiting on a person.

Three kinds cover what a response can be, and the kinds are content the deployment defines rather than anything blizzard
enforces. **Remediate** does the cleanup the findings describe. **Prevent** fixes the source so the inflow stops — the
missing standard, the exemplar spreading the pattern. **Mechanize** retires the judgment altogether by moving the check
into a lint rule or a gate. Prevention is usually the highest-leverage of the three and remediation the most obvious,
which is exactly why recording the kind is worth it: an axis whose accepted proposals are all remediation is being
weeded while its source runs untouched.

Proposals attach to the axis and go no further on their own. They are not chunks and they do not become work without a
person, because that judgment is not the fleet's to make — it has said what it thinks, and the decision belongs to
whoever reads it. Two verbs close one out. **Passing** leaves the record that it was considered and declined, so a later
run's cross-reference does not raise it again as though it were new. **Accepting** creates a self-sourced work item
linked to the proposal in the same act.

That link is not a nicety; it is the entire measurement story. Findings resolve because work lands, and knowing which
work resolved which findings is what separates a real inflow-and-outflow from two unrelated counts. So acceptance is a
verb inside blizzard even when the item's body is drafted outside it — paste the text in, but let the act that creates
the item be the act that carries the link.

## What the store buys

Because findings are rows, a trend is a query rather than an errand. The findings created in a week are that axis's
inflow and the ones resolved are its outflow, and an axis whose inflow outruns its outflow week after week is losing
ground in a way no single run would show. Where a finding records the commit that introduced the code it objects to —
best effort, since `blame` is defeated by a reformat and has no honest answer at all for a rule that went stale because
the code around it moved — the same data cuts a second and sharper way. Findings attributable to the last month's
commits, held against findings attributable to older code, separate prevention from remediation: a garden whose
recent-code findings are falling while its total holds flat has stopped the inflow and simply not drained the backlog
yet, which a bare count of open findings would report as no progress at all.

Nothing re-verifies a finding on a schedule, but cross-referencing re-observes as a byproduct — a finding inside a run's
scope is either seen again and transformed to say so, or absent and transformed to say that. Findings outside the scope
keep their last word. So how recently a finding was observed is free exactly where it is meaningful and honestly stale
where it is not: one seen this morning in a weekly-swept package carries real confidence, while one last observed six
weeks ago is telling you nothing has looked at that ground since — its own useful fact, and one no re-reading was paid
for.

Outflow keeps a human in it throughout. A finding leaves the live set when the work behind it lands, when someone
withdraws it, or when someone accepts a run's report that it no longer reproduces — never on the pass's word alone.

This is scope `epic:self-sourced-work` did not cover. That epic built the lane for turning opinions into work. Findings,
their sets, and proposals are new durable hub entities, with their own endpoints, a CLI verb letting a run read its
axis's live findings, and a place in the gardening tab where a lineage's trend can be seen rather than merely recorded.

## The pass machinery

- **The gardening tab.** Gardening lives in its own tab in the hub webapp rather than a corner of the board. The board
  watches work in flight; gardening is a standing discipline an operator sets up once and tunes over months, and the two
  deserve separate surfaces. The tab is the home for the whole of that setup — the routines a deployment has declared,
  the graphs behind them, each axis's live findings, and the proposals waiting on a decision.
- **Garden routines, authored in the product.** A routine is a named, hub-stored strategy: a name, the axis its findings
  are attributed to, the graph it runs, a default scope, and model/effort defaults. Creating and editing them is the
  capacity this epic adds — routines are authored in the gardening tab and through CLI verbs, by the operator, at the
  moment a project has grown enough to need one. A cadence field is the end state, deliberately absent for now.
- **Instantiation, with options.** `blizzard hub routine run <name>` (and an action beside the routine in the gardening
  tab) mints a fresh work item from the routine, ingests, and promotes it in one act — manual kickoff is itself the
  deliberate human trigger. The run takes options: a **scope** — one of the names its routine's strategy declares,
  defaulting to the routine's own — a **mode** (full sweep, or delta — see below), and a free-text **charge note**
  appended as a "This run" section. Since the consumer is an LLM reading prose, parameters are concatenation, not a
  substitution grammar. Machine state stays out of the preambles: the running pass fetches its live context, and its
  axis's live findings, through the API.
- **Lineage.** Every run carries its routine's name and its scope — on its work item, never on its chunk — and so does
  every finding set, finding, and proposal that run delivers. Lineage makes runs siblings, and it is the query key both
  cross-referencing and the trend turn on.
- **The convergent standing pass.** A run does not start from nothing. It reads the live findings its lineage has
  accumulated, then delivers a delta against them: new drift as an addition, a finding still reproducing as a
  transformation recording that it was seen again, a finding it looked for and could not find as a transformation saying
  so. Convergence is **scope-aware**, enforced twice over: the run fetches its scope's bucket rather than the whole
  axis, so findings recorded under other scopes are never in front of it to absolve; and the delta shape means silence
  is never a claim, so ground inside the bucket that the run did not actually reach keeps its last word too.
- **Fan-out and the gate, both the graph's.** Neither is machinery this epic adds. When scope exceeds one context it is
  the survey node's preamble that says to partition — packages, features — and the pass spawns sub-readers through
  whatever facility its harness gives it; a chunk travels one node at a time and what a session does inside a node is
  its own business, so blizzard cannot tell a fanned-out survey from a monolithic one. What the platform does owe
  fan-out is on the way out: a node may declare several lists, and delivery mints a set per list, which is what keeps
  the partitions distinguishable afterward. Whether a human gate sits anywhere in a run is likewise not this epic's to
  decide: a runner imposes one on any node by name from its own config, and blizzard's own routines run without one.
  Nothing in a garden run needs a person mid-flight — findings are evidence that lands either way, and proposals attach
  to the axis and wait, so every decision worth a human is available afterward, batched, in the gardening tab.
- **Mechanization is the metric, not the mechanism.** Which of a pass's judgments have earned a deterministic check is
  the strategy's call, and the propose node owns how a run makes it. What this epic owes that call is that it be a
  *query* rather than a memory: a finding's class, its recurrence across runs, and whether a human ever passed on the
  proposals it raised are facts the store keeps, so "this class recurs and nobody overrides it" is answerable without
  anyone having tracked it by hand. The success metric follows from that data — a garden that shrinks its own LLM
  surface over time, judgment migrating into reflexes until the passes spend their context only on what genuinely needs
  a mind.

## Garden states and target scales

A pass judges by whatever its prompt names, and that one fact makes its behavior proportionate to the condition it finds
the garden in:

- **Healthy garden**: silence is the success state and it must be cheap. A converged run delivers zero items, a few
  still-fine confirmations, and a measurement on its run row — the flat trendline is what upheld gardening looks like as
  evidence rather than a feeling.
- **Overgrown garden** (a run finds thousands): read it as a signal before treating it as a workload. A tended garden
  does not routinely produce a thousand findings, so a run that does has almost certainly met a **shift in grading** — a
  standard newly authored, or newly sharpened, that the existing code was never written against. That is a real and
  useful thing for a pass to surface, and it is not what gardening is for: the response is upstream, not a thousand
  cleanups. So — **stop the inflow first** (the rule that would have prevented the pattern, or the exemplar spreading
  it); **say the number honestly** (the count is the finding that matters, and a pass reports it whether or not it
  enumerates every instance behind it); **aggregate into campaigns** (a remediate proposal per area, naming the findings
  behind it and sized as an executable chunk of work, so the inventory stays queryable without any of it reaching the
  backlog); **propose boundedly, converge the rest**. A pass never files a thousand items — the backlog is a human
  triage surface, and per-instance filing confuses inventory with work. Draining a backlog that large is a different
  problem than tending a garden, and if these passes keep exposing it, it earns its own machinery rather than this one.

Scale is handled by the run options and one economy:

- **Tiny targets**: the deployment declares fewer routines, or one merged one — a routine per axis is a deployment's
  choice, never the machinery's mandate.
- **Big targets** (one context holds it): the default — whole-scope runs, faceted fan-out when useful.
- **Gigantic targets** (no context holds it): covered by rotating **scoped runs** under one lineage — scope-aware
  convergence is what makes sharding honest — with fan-out inside each run. And after one full sweep, **delta mode** is
  the steady state: weeds grow where change happens, git answers what changed since the lineage last ran, so a run
  examines the delta against its axis's live findings, and cost scales with change rate rather than repo size. The full
  sweep becomes a rare deliberate act — a new routine, or a changed standard, which re-opens judgment on unchanged code.

## The end state

Autonomous recurrence — the eval that runs every 7 days whether or not anyone remembers it — is one small addition to a
shape that will not change: a cadence field on the routine and a hub sweep that runs any routine whose interval has
elapsed and whose previous run is not still live. The convergent pattern is what makes the schedule safe to turn on: a
run that reads its axis's live findings and delivers a delta against them cannot duplicate what it already recorded, no
matter how often it fires, so automation is a trust decision per routine, not new machinery. It stays a judgment rather
than a threshold: no run count and no override rate earns a routine its cadence, because the person reading the
proposals is the one who knows whether they are worth waking up to. Until then, exploration is the mode: edit the
charge, run it again, watch what it proposes — with the findings backlog persisting across every experiment.

## What this epic is not

Not a scheduler yet: manual kickoff is the exploration mode, and the cadence ships when a pass has earned it. Not edit
authority: a pass never prunes anything directly — everything it touches changes only through promoted chunks riding
normal delivery. Not a definition of what a weed is: blizzard supplies the machinery and the deployment supplies the
strategy, so nothing built here decides what any project ought to prune. Not analytics or mutation testing: a strategy
consumes their evidence where it exists, and works from judgment and git where it does not.

## Open questions

- Whether a routine can be triggered by a landing, rather than only by hand or a clock — a pass reading one delivery's
  diff while it is one commit old, instead of a scope of code on a cadence. It is a machinery question, a third trigger
  beside the two this epic builds, and the sweep decides it: what a cadence run catches, and how much of that a human
  reviewer had already caught on the way in, is the evidence for whether a per-delivery trigger earns its cost.
