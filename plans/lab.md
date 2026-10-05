---
epic: lab
refinement: scaffolded
slices:
  - name: bench
    status: horizon
  - name: experiment
    status: horizon
  - name: lab-runner
    status: horizon
  - name: specimen
    status: horizon
  - name: verdict
    status: horizon
  - name: lab-board
    status: horizon
  - name: seeded-defects
    status: horizon
  - name: evidence
    status: horizon
---

# Plan — `epic:lab`

[`persona:harness-engineer`](../charter/personas/harness-engineer.md) changes one thing, runs the fleet, and reads the
results. Today that last step is mostly faith. A plan-review prompt is rewritten, a reviewer moves from one model to
another, a convention is added to the harness every worker reads, and the evidence that it helped is a few nights that
felt better than the week before. Those nights ran different work on a moving codebase, with every node of every chunk
free to vary at once, so a better plan, a luckier build, and a stricter reviewer all arrive tangled in the same outcome.
The engineer wants what any experimenter wants: two runs that differ in exactly one respect, and enough of them to tell
a real difference from the noise.

The lab gives them that, and it does it at the grain where the answer actually lives: the node-step, not the chunk. A
whole chunk run twice is a poor experiment. The plan diverges in the first hour, and everything downstream inherits the
divergence, so a question about reviewers is drowned by differences in plans the reviewers never chose. A lab instead
holds everything upstream of the question fixed, runs only the stretch of the graph the question is about, stops each
run at the boundary of that stretch, and compares what came out. It is cheaper by the same stroke: a trial costs one or
two node-steps, not a night.

## Not yet: a winter lab, by hand, first

None of this epic is being built yet. Before blizzard grows experiments, trials, and a board of its own, the lab has to
earn them by being useful without them. The first step happens outside blizzard entirely: a **winter lab**, a winter
workspace whose feature environments each hold a whole other workspace, pinned to one commit, with a lab extension that
tells the agents working there how to design an experiment, turn a graph node into a bench, run trials in paired waves,
and read the result. The harness engineer drives it by hand, at a terminal, on a few real questions — the reviewer that
seems to have gone soft, the convention that may or may not be pulling its weight, the cheaper model someone wants to
try on plan-review.

That manual lab is the proof this epic waits on. If a handful of experiments run by hand change a decision someone would
otherwise have made by feel, the case for bringing the lab into the fleet makes itself, and the slices below are built
knowing which parts of the work were worth automating and which were not. If they change nothing, the epic has cost a
few afternoons instead of a quarter. Either way, what blizzard eventually builds is shaped by an experimenter's actual
habits rather than by this plan's guesses about them.

The manual lab needs more of winter than winter has today: a workspace hosted inside another workspace's environment,
with its own port band, the identity its parent hands it, its context sealed from the lab around it, and the whole
definition frozen to exact commits and fingerprinted so two copies are provably the same. Winter's nested-workspaces
work carries all of it, and each piece is useful to an ordinary workspace too. The lab can be run before that lands,
with a script doing what winter will later do, which is how its first experiments should be run.

Some of what the lab must hold to is already clear, because a lab that gets it wrong produces confident nonsense:

- **Isolation is proven, never assumed.** A trial's session runs inside the lab's directory tree, and a harness that
  walks upward for instruction files will find the lab's own unless told otherwise. Before an experiment's first wave,
  the lab asks a trial session which instruction files it loaded and refuses to run if any lies outside the arm.
- **A trial that never reached its end state measured nothing.** A reviewer denied the command it needed reports that it
  could not review; scored as a miss, it reads as a reviewer that missed the bug. Such a trial is invalid, and the lab
  says why.
- **A bench runs under the fleet's own permissions.** Agents chain shell commands that no narrow allowlist anticipates,
  and a bench held to a tighter posture than the fleet stops measuring the node it was cut from. What guards an arm is a
  fingerprint taken before and after each trial; a trial that changed its own arm is void.
- **The control is run against itself.** Two identical arms disagree more often than intuition expects, and that
  disagreement is the ruler every other difference is read against.

## The lab is a graph transformation

The lab's central idea is that an experiment is a graph, derived from the one under study. The engineer points at an
existing graph, say `adv-dwf`, and names the **bench**: the node-steps under study and the loop between them. For a
question about planning, the bench is `plan`, `plan-review`, and the revise edge that sends a plan back for another
pass. The lab transforms the source graph into a lab graph around it:

```text
adv-dwf:  plan ⇄ plan-review → build → verify → … → deliver → … → done

lab:      [seed] → plan ⇄ plan-review → ◼ end state (would-have-built)
```

Three rewrites carry the whole transformation. The bench is kept verbatim, with its nodes, prompts, judgements, and the
edges among them, so what runs is what the fleet really runs. Everything upstream of the bench is replaced by a seed:
the artifacts those nodes would have produced, supplied from a recorded run rather than produced again. And every edge
leaving the bench is cut and capped with an **end state**, a terminal that records which way the work was about to go
and what it was carrying, then stops. `build` and `deliver` are never run; they are hijacked into end states named for
what would have happened next. An end state either returns the trial to the lab for analysis, or segues into a lab graph
of its own, an evaluation graph whose nodes judge what the bench produced.

The bench can be any connected stretch. A single node is the narrowest case: `review` alone, fed one fixed diff, judged
on what it finds. A wider bench such as `build ⇄ verify` answers a different question, and the lab neither knows nor
cares which question is being asked; it knows only which nodes to keep, what to seed, and where to cut.

**Arms** are overlays on the bench. An arm names what differs from the source graph and nothing else: a declared
session's model, effort, or harness, a node's prompt or an edge's addendum, a profile once `epic:worker-profiles` gives
settings a name. Those are the overlays the hub can mint into a graph. An arm may also name a workspace: the
conventions, skills, agents, and tooling a worker finds around it when its session opens, which are often the very thing
under test and which the hub does not own
([Project workspaces and lab workspaces](#project-workspaces-and-lab-workspaces)). Every arm of an experiment is minted
from the same transformation, so two arms differ by exactly their overlays, and an arm with no overlay is the control.

A lab graph is minted like any other graph, under a name of its own, so it never becomes the newest `adv-dwf` and no
chunk ever drifts onto it. Lab graphs never follow a newer mint, never land, and never turn the work items a bench node
proposes into real backlog; an end state is the end of a trial, not a delivery. Because the transformation is
mechanical, the lab can also refuse at mint what it cannot isolate: a bench whose nodes read an artifact nothing seeds,
or a bench node that resumes a session begun upstream of it, as `pre-push` resumes the conversation `build` started in
`adv-dwf`, is named and resolved before a single trial runs rather than discovered as a confound afterward.

## Project workspaces and lab workspaces

A runner brings its own workspace, and the hub never dictates which one or at what commit. That is deliberate and it
stays true: a project workspace is long-lived and shared, many chunks pass through it one after another, and a fleet
that cut a fresh, pinned workspace for every chunk would spend more on isolation than on work. Ten runners with one
environment each would satisfy a purist and waste a machine.

The lab is the one place where a workspace must be pinned, because there the workspace is the thing being measured. So
the lab brings a different kind of workspace: a **lab workspace**, a meta-workspace that holds many workspaces, one per
arm, each cut from a versioned definition — the workspace's commit and the exact commit of every extension it installs —
and initialized in place with its own environments for trials. It is the same move winter already makes one level down:
where a project workspace is one workspace with many feature environments, a lab workspace is one lab with many
workspaces. The lab workspace is itself versioned like any workspace, and the workspaces inside it can be any winter
workspace whose context is complete in its own repository.

A runner therefore offers one of two things: a project workspace, for chunks working on a project, or a lab workspace,
for trials. Project runners never claim trials, so an experiment can never pin, disturb, or borrow the environment real
work runs in, and a trial only ever runs where its arm's workspace can be materialized exactly. An experiment names each
arm's workspace definition by a reference the hub stores and never interprets; the lab runner resolves it, materializes
it, and reports back a fingerprint of what it primed the trial from, so the hub records which definition every trial ran
in without ever reaching into the runner. Two arms meant to be identical fingerprint the same, and a trial whose arm
changed while it ran is set aside.

A lab workspace is also a **clean room**, and says so only once it has shown it: before an experiment runs, the lab
certifies that a trial session loads the arm's instructions, skills, and agents and nothing else — not the lab's own,
not a stray file in a directory above it, and not a user-level setting that belongs to no arm. User-level harness
configuration is pinned rather than erased, since the fleet's own workers run with one; which configuration a lab runner
pins is part of the definition an arm can vary.

## How an experiment runs

A lab graph describes one traversal of a bench, and it should never describe more. Running a bench twenty times is not
something the graph does; it is something the experiment asks for. A graph that looped twenty times would run its trials
one after another on one runner, pile twenty attempts into the same artifact series where no reader could tell them
apart, and turn the count of trials into a property of an immutable definition, so that asking for five more would mean
minting a new graph. The graph stays the shape of a single trial, and everything about how many trials, of what, and in
what order belongs to something above it.

That something is the **experiment**: a record of its own on the hub, a sibling to the routine rather than a kind of
chunk or a kind of work item. It names a hypothesis, the bench and the lab graph minted for each arm, the specimens to
run them against, how many times to repeat each pairing, the evaluation each trial's end state feeds, and the budget it
will stop itself at. An experiment is launched deliberately, the way a routine's run is, because it spends money the
moment it starts.

Each **trial** is an ordinary chunk, and that is the most important choice in the epic. A chunk already has everything a
trial needs: a runner claims it under a lease, it is fenced against a stale attempt, it records every movement and every
token as facts, its transcripts are shipped home, and a crashed runner hands it to another. Nothing in that machinery
has to learn about experiments. What marks a trial is where it came from: it carries its experiment, arm, specimen, and
replicate, and it wraps no backlog item, so it can never close one, never collide with the live chunk working the same
item, and never be mistaken for work that owes a landing.

The experiment is the conductor the graph model does not have. No chunk can wait on many, so the experiment does the
waiting. It mints its trials in waves, each wave holding every arm of the same specimen together, so the arms run side
by side through the same hour of provider weather and against the same state of the fleet. As trials reach their end
states it mints whatever the comparison needs next — a judging chunk that receives two trials' outputs once both exist,
a person's blind choice on the lab board — and between waves it reads what it has. When the difference between the arms
is clear, when it is clearly smaller than the drift between two identical runs, or when the budget is spent, it stops
minting and says which of the three ended it. Twenty runs is a ceiling the engineer sets, not a quota the lab must fill.

How those twenty are spent matters as much as how many there are. Twenty repetitions of a single specimen say a great
deal about that one piece of work and nothing about the next; ten specimens run twice each say less about any one and
much more about the fleet. The experiment spends repetitions where they are informative: enough of the control against
itself to know how noisy a single trial is, and the rest spread across specimens, because generalizing beyond the work
that happened to be chosen is the point.

Trials travel the same ready queue as everything else, ranked in the group that yields to real work, which is what
`epic:urgency`'s `background` group already promises, but only a runner that brings a lab workspace claims one. A
separate queue would need claiming rules of its own; a runner that declares what kind of workspace it offers needs only
the claim it already makes. An experiment declares how many of its trials may run at once, so it can never crowd its lab
runners, and an arm that names a harness reaches only the lab runners that offer it, by the same harness choice every
declared session already makes.

Twenty trials would bury the operator's board, so they do not appear on it. The board shows an experiment as a single
card, and the trials behind it live on a board of their own, the **lab board**, where the question being asked is
different. The operator's board asks what the fleet is doing; the lab board asks what the fleet has learned. It shows
each experiment's design and the overlay each arm carries, a grid of specimens against arms filling in as trials reach
their end states, the running estimate and the uncertainty around it, the budget spent and remaining, and the queue of
blind comparisons waiting for a person's judgement. Any cell in the grid opens the trial as the ordinary chunk it is,
transcripts and all.

## The bench slice

The first slice is the transformation itself: naming a source graph, a bench, and a set of arms, and minting the lab
graphs that result, with end states capping every exit and a seed standing in for everything upstream. It is useful
before anything else in this epic exists. With a seed written by hand, an engineer can start a trial chunk on each arm
by hand and read the end states side by side. Paired with `epic:demo`'s mock harness, a lab graph can be run for free,
which makes the transformation itself testable without spending a token.

## The experiment slice

The second slice is the experiment as [How an experiment runs](#how-an-experiment-runs) describes it: the hub record
that names arms, specimens, repetitions, evaluation, budget, and concurrency; the trial as a chunk that carries its
experiment and wraps no backlog item; and the conductor that mints trials in paired waves, mints the comparisons that
need two finished trials, and stops on a clear answer, a clear non-answer, or an empty budget. It is what turns the
bench slice's hand-started trials into a measurement.

## The lab runner slice

The third slice is the runner that brings a lab workspace, as
[Project workspaces and lab workspaces](#project-workspaces-and-lab-workspaces) describes it: a runner that declares it
offers a lab workspace and claims only trials, materializes each arm's workspace definition on demand, certifies the
clean room before an experiment's first wave, and reports the fingerprint of every trial's arm as a fact. It is the
manual winter lab with the fleet's claiming, leasing, and fact log around it, which is why the manual lab comes first.

## The specimen slice

A seed written by hand answers only the questions its author thought to ask. The fourth slice draws seeds from real
work. A **specimen** is a recorded chunk captured at the moment it entered the bench: the artifacts it held, the commit
its worktrees stood on, and the work items it wraps. The engineer collects specimens from the fleet's own history (the
chunks that delivered, the chunks that escalated, the ones that looped five times through review) and runs every arm of
an experiment against every specimen. The trials pair naturally: each specimen meets each arm, so the hard specimens are
hard for every arm alike and their difficulty cancels out of the comparison.

A specimen replays the past, and the past must stay sealed. A worker re-planning a chunk that already delivered must not
be able to read the answer: the worktree is cut at the specimen's commit, with nothing later reachable from it, and no
trial sees another trial's branches, findings, or artifacts. A trial never wraps its work item the way a live chunk
does, so a specimen of work that is still in flight can be studied without disturbing it. Where a trial asks a human a
question, the specimen's recorded answer is replayed to every arm alike, so a person's mood on a given evening never
decides an experiment.

The fleet's own history is what makes this compound. Every chunk that delivers is a specimen with a known good outcome
attached, the change that actually landed, and the collection of them becomes a benchmark of the project's own work that
grows every night.

## The verdict slice

The fifth slice reads the trials. Some of what it reads the fleet already records as facts and costs nothing to collect:
how many times a gate sent work back, how many retries a node spent, whether a trial escalated or asked a question, the
tokens and time each node-step took, and which end state the trial reached. A plan-review that waves a weak plan through
and a plan-review that loops it four times leave very different traces, and the count of loops per gate may be the
single most honest number the lab produces.

Other readings need judgement, and the lab supplies it by graph, so a judge is a node like any other and can itself be
put on a bench later. An evaluation graph receives what a trial's end state carried and judges it: the faceted reviews
and garden axes the fleet already holds itself to, run over each arm's output; a blind pairwise comparison that sees two
outputs with the arms hidden and the order swapped, judged by a model from a family neither arm used; and, where the
question deserves it, a person shown two results on the board and asked which they would merge. Their choices accumulate
into the measure the automated judges are later calibrated against.

The verdict is computed by the hub from the trials' facts, never stored as a conclusion an agent wrote. It reports
effect sizes with their uncertainty, not a winner, and it is willing to say *inconclusive*. Two disciplines keep it
honest. Before an experiment's arms are compared to each other, the control is compared to itself, to measure how far
two identical runs drift apart; a difference smaller than that drift is not a difference. And cost is reported beside
quality rather than folded into it, because "nine-tenths as good at a third of the price" is a decision the engineer
should make with both numbers in view.

## The lab board slice

The sixth slice is the [lab board](#how-an-experiment-runs): experiments collapsed to a single card on the operator's
board, and a surface of their own where an experiment's design, its grid of trials, its running estimate, its budget,
and its queue of blind comparisons are read. It comes after the verdict because it renders what the verdict computes,
and before the slices that follow because seeded defects and evidence are both read there.

## The seeded-defects slice

The hardest thing to measure about a gate is what it misses, because in ordinary work no one knows what was there to
catch. The seventh slice manufactures that knowledge: the lab plants known defects into a specimen's artifacts, a bug in
a diff on its way to `review` or a missing requirement in a plan on its way to `plan-review`, and measures how often
each arm catches them. It is the same move `epic:mutation-testing` makes against a test suite, made against the fleet's
own reviewers, and it yields a number no one has today: how often the review gate catches a real bug.

## The evidence slice

The last slice closes the loop the lab exists for. The fleet already proposes changes to itself: a retrospective names a
recurring failure, a garden pass drafts a proposal. Today such a proposal is judged on argument alone. Here a proposal
to change a prompt, a graph, or a convention can carry an experiment run on the lab, so the person approving it reads
evidence beside the case. And once a change is adopted, the lab can keep watching it: a small share of live chunks gets
a shadow trial on the previous configuration, never landing, so a change that helped in the lab and quietly hurts in the
field is noticed by the fleet before its owner notices it by feel.

## What the lab is not

The lab does not choose configurations on the operator's behalf. Routing live work across arms by their running results
is a natural next step and a different product; the lab produces evidence, and a person decides what it is worth. Nor is
it a general benchmark harness for models: its subject is always this fleet, its graphs, and its project's own work,
which is precisely the evidence that no public benchmark can supply.

## Open questions

- How a seed reaches a bench. A node's inputs today arrive only with a judged step, so there is no way to place a chunk
  mid-graph with its upstream artifacts already present. Baking the seed into the lab graph as graph-scope artifacts
  works within the model as it stands; a first-class way to seed a chunk at a node may be worth having for its own sake.
- How a bench shares sessions. A bench node that resumes a session begun upstream inherits a conversation the seed
  cannot reproduce. Whether the lab starts every bench session fresh, replays the upstream transcript, or refuses such a
  bench decides how faithful a trial is to the graph it came from.
- What a chunk that wraps nothing is. A chunk today wraps one or more work items, and a trial deliberately wraps none.
  Whether a trial is a chunk with an empty work list, or wraps a hub-authored item the experiment owns that never
  closes, decides how much of the chunk's own rules bend for the lab.
- When the conductor stops. Stopping on a clear answer is a statistical rule as much as a product one, and stopping
  early on a lucky streak is the oldest mistake in experimentation; the rule must be fixed when an experiment launches,
  never chosen after its numbers are in.
- Whether the arms of a harness comparison can ever be equal. Two harnesses differ in their tools and system prompts as
  well as their models, so "Claude against GPT" through two harnesses is really two bundles, and the lab should say so
  rather than report it as one variable.
- How a lab runner learns an arm's definition. The reference the hub stores might name a definition the lab workspace
  already holds, or a workspace repository and the commits to pin it to, which the runner materializes on first use. The
  first makes arms something an engineer prepares; the second lets an experiment ask for one on its own.
- Whether two workspaces can be compared across two hosts. Arms on different machines differ in their load, their
  subscriptions, and their user-level configuration as well as their workspace. One lab workspace holding every arm is
  the clean comparison; a comparison across hosts should be reported as the bundle it is.
- How judges are kept honest. A judge prefers longer answers, the first answer shown, and answers from its own family,
  and a fleet tuned against a judge learns to please it. Rotating judges and keeping people in the sample are the
  defenses; how much of each is the open part.
