# Milestones

What users will be able to do. A milestone is a destination stated in the user's terms, and it comes first: you declare
where the product must reach, then ask what work the journey requires — the milestone demands its epics, never the other
way around.

| Milestone                      | What users will be able to do                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `milestone:polyglot`           | Run the fleet on the coding harness of their choice — Claude Code, Codex, or OpenCode, first-class and mixable by node — with the safeties on: no worker runs with permissions dangerously bypassed.                                                                                                                                                                       |
| `milestone:projects`           | Run every project from one fleet: a single hub hosting many projects and the sources they draw from, and a single runner host per machine working all of them — a workspace for each project or set of projects, not a stack per project. Each project lands and deploys its finished work its own declared way.                                                           |
| `milestone:hardening`          | Decide for themselves how the fleet behaves: every operational constant theirs to set, what each runner will take and when and at what rate theirs to declare, a provider outage ridden out rather than slept through, nothing growing without end underneath them, and every question about the platform answerable without cloning it.                                   |
| `milestone:human-in-the-loop`  | Stop being the wire between the fleet and everything it needs: it reaches them wherever they are when a decision is genuinely theirs, and settles CI's verdict itself when it is not.                                                                                                                                                                                      |
| `milestone:mobile`             | Carry the fleet in a pocket: watch the night, answer a question, and unblock a chunk from a phone, through notifications that arrive the way the phone's own do.                                                                                                                                                                                                           |
| `milestone:project-management` | Assemble what the fleet works on from inside blizzard — browse the backlog, take many items at once, and shape them into chunks — instead of handing over ids one at a time.                                                                                                                                                                                               |
| `milestone:future-state`       | Look inside the fleet while it works and steer it without stopping it — a live environment opened from the board, a word to a running chunk — over a fleet that keeps its own schedule, hears from the systems that notice trouble first, bends to a shop blizzard's authors never saw, and settles by experiment rather than by feel which way of working serves it best. |

## `milestone:polyglot` — any harness, with the safeties on

Blizzard was built around a harness seam — spawn, resume, verdict — precisely so that no single coding agent would ever
be load-bearing. The seam now has two occupants — Claude Code and OpenCode, mixable by session lineage on one runner —
but Codex is still outside it, and unattended workers run with broad tool approval. This milestone completes the harness
story in both directions. Breadth: Codex joins the same seam, chosen per fleet or mixed by node the way models already
are. Control: each runner's operator brings a set of native harness configurations alongside the runner's own hooks and
headless rules, and chooses one autonomy level that maps to each harness without leaving a headless run stuck on a
prompt. The name is what the fleet becomes: polyglot, fluent in more than one harness and able to set terms for each.

Through the charter's people: the harness engineer runs the same chunk through two harnesses and compares the runs on
cost and quality, because harness choice has become a tunable rather than a fact of the platform. The application
architect can bring harness extensions and tool rules into unattended work without editing a generated settings file.
The operator chooses normal, auto, or dangerous once for the runner, knowing each harness maps that choice to its own
capabilities and protected branches remain the forge's to guard, not the agent's to respect on a bad night.

| Epic            | Slice           | Status      |
| --------------- | --------------- | ----------- |
| `epic:adapters` | opencode        | delivered   |
| `epic:adapters` | codex           | horizon     |
| `epic:security` | worker-security | in-progress |

## `milestone:projects` — one fleet, every project

Today the platform is single-project by silent assumption: one hub, and every source feeding it belongs to blizzard. The
operator who wants the same machinery working winter — or celestial frontier — stands up a second hub, a second runner,
a whole second stack on the same desk, and none of the stacks know of each other. This milestone makes *project* a
first-class idea: a grouping of both what to do and who does it. One hub hosts many projects drawing on shared sources —
one Jira can feed all of them — and a piece of work carries the project it was ingested into from ingest to landing.

The machine is rearchitected to match. Its runner host — the one daemon a machine runs — stops being the extension of a
single workspace and becomes a host of many: it configures local workspaces, each serving one project or several whose
repositories it holds together, and each works for its projects through whichever of the host's runners serve them.
Three projects on one laptop means three workspaces and one host — never three installations, and never three of
everything above them. For the operator, a desk full of stacks collapses into one: queue work against any project, watch
all of it on one board, and slice the view to a single project when only that one matters. That host is the one
`epic:runner-host` builds for `milestone:hardening`, and this milestone's host work waits on it; until then, a runner
serves the projects whose repositories its one workspace holds, and a project worked apart gets a runner of its own.

None of it could be grouped while it lived in a file on the hub's machine and its tokens in the hub's environment, where
adding a source or rotating a token meant a redeploy and nothing could belong to one project rather than another. The
first step moved that configuration into the hub's store, behind a secret store that takes a credential once and never
hands it back, where the operator changes it while the hub runs and the grouping that follows has something to own.

A project organises one operator's world; it does not partition the hub. Graphs stay a shared library and the board
still shows the tenant's whole fleet, so the boundary that holds for everything is a level above projects: the tenant.
One hub can host several wholly separate worlds, each with its own graphs, projects, runners, and board, and none aware
of the others. The first to need that is blizzard's own test suite, which wants hundreds of tests running against one
hub, each in a world of its own. Tenancy threads through the same store, API, and runner seams that projects reshapes,
so the two are designed together and built back to back, tenancy first, so that every project is born inside a tenant
rather than one reopening what the other just finished.

A project also decides where its finished work goes and how it gets there. Repositories belong to the tenant and each
project links the ones its work may land in — a shared library can serve two projects — while each repository says how
work lands in it and how it runs once landed. One project merges its own work while another waits for a human to merge
every pull request; one repository fast-forwards while its neighbor rides a merge train; and a chunk that touched three
repositories deploys all three in the order they depend on one another, proving each healthy, or rolling it back, before
the next one moves. Delivery and deployment stop being one fleet-wide habit and become each project's own declared
method, carried out deterministically, with an agent's judgement called in where the project asks for it.

| Epic                       | Slice  | Status    |
| -------------------------- | ------ | --------- |
| `epic:live-config`         | full   | delivered |
| `epic:multi-tenancy`       | hub    | horizon   |
| `epic:projects`            | hub    | horizon   |
| `epic:projects`            | runner | horizon   |
| `epic:multi-tenancy`       | runner | horizon   |
| `epic:runner-host`         | runner | horizon   |
| `epic:projects`            | host   | horizon   |
| `epic:advanced-delivery`   | full   | horizon   |
| `epic:advanced-deployment` | full   | horizon   |

## `milestone:hardening` — the platform stops guessing on your behalf

A young platform is full of numbers somebody chose once. How long a lease lives, how many times a node retries before it
escalates, how long the reconciler sleeps between sweeps — each of them was settled during a build, by someone who was
not running this fleet, on this machine, against this repository. They are usually close enough to right. When one is
wrong it is wrong in code, and correcting it costs a change, a review, and a release.

The harness engineer feels that most sharply, because their whole craft is comparison: run the same chunk twice with one
variable moved and see what it cost. A constant they cannot reach is a variable they cannot move, and a question they
are simply unable to ask.

The same presumption runs past the numbers. A runner today takes any work, at any hour, at whatever it costs — the
platform assuming an operator who wants everything immediately and at any price, and offering them no way to say
otherwise. Two levers answer it. The first is pacing, which is spend understood as a rate rather than a ceiling: a
weekly allowance spent evenly is an allowance that lasts the week, so a runner six days into seven with only a day's
budget left should decline to start anything new, and a machine its owner wants back during working hours should stand
down on a schedule rather than on a reminder. The second is routing. Work carries tags — a type, a risk, a project — and
a runner declares which of them it will accept and which it refuses, so the operator who wants one machine kept to
small, safe fixes says so once instead of watching the queue.

The board has the same problem in a different material. Every small surface it grows — a form, a picker, a confirmation
sheet — is invented, styled, and maintained by hand, and the count climbs with every epic that touches it. Hand-built
parts drift: two dialogs written six months apart agree on nothing, and the operator learns each of them separately.
Assembling the next one from a toolkit is faster to write, but the reason it matters is that it is quieter to use.

Two assumptions remain, and both concern what happens when things go badly or go on for a long time. The fleet presumes
its model provider is reachable, so an outage or a rate limit at one in the morning becomes a queue of failures and a
wedged morning, when what the operator wanted was a fleet that waited and picked up where it left off. And it presumes
storage is free: every fact, transcript, artifact, and measurement is kept forever, which is comfortable for a year and
then arrives without warning as a daemon that will not start.

The last assumption is about the reader. Blizzard is public and installable, and its README is the entire published
surface — so the platform presumes that anyone with a question past the first one is willing to clone it and read. The
operator standing a fleet up, the graph author deciding what a node may declare, and the engineer weighing whether
blizzard fits their shop each arrive with a different question, and each pays the same entry fee to answer it. A
published documentation site is what a platform owes a reader who has not committed to it yet.

The platform's own suite makes the same presumption about time. Every chunk the fleet lands waits for the gate, and the
gate has grown from a minute's check into ten minutes and more. The time is not spent on proving more: it goes to work
repeated per test, tests run one at a time that could run side by side, and waits measured by a clock rather than by the
system. The suite's upper tiers also presume their implementation. They describe how the running platform behaves, yet
they reach into its Python to say so, which leaves them unable to hold a rewrite to the same promise. Faster tests, and
tests that describe the platform from outside, are both hardening of the ground everything else stands on.

The code makes one presumption of its own: that whoever reads it already knows where everything is. Most of blizzard is
now written by agents meeting it for the first time, and they find a behavior from its name only when the layout
announces it. A hub domain of sixty-odd modules in one directory, and a runner sliced by what its code does rather than
what it is about, both ask the reader to carry a map the code no longer draws. Giving every concept one package, every
package a one-way place in the order, and every rule a home on the model it governs is the same hardening turned inward
— and it holds only if the build refuses the drift, not if someone remembers to.

Hardening is what a platform does after it works. Little of it is a new capability. Almost all of it is an assumption
the platform made on the operator's behalf, handed back to them as a decision.

| Epic                       | Slice            | Status    |
| -------------------------- | ---------------- | --------- |
| `epic:runner-host`         | runner           | horizon   |
| `epic:runner-host`         | hub              | horizon   |
| `epic:throttling`          | full             | horizon   |
| `epic:tagging`             | full             | horizon   |
| `epic:resilience`          | full             | horizon   |
| `epic:retention`           | full             | horizon   |
| `epic:config`              | model-resolution | horizon   |
| `epic:config`              | sweep            | horizon   |
| `epic:ui-toolkit`          | full             | horizon   |
| `epic:documentation`       | full             | horizon   |
| `epic:test-optimization`   | full             | horizon   |
| `epic:test-architecture`   | full             | horizon   |
| `epic:test-shared-service` | full             | horizon   |
| `epic:architectural-sweep` | full             | delivered |

## `milestone:human-in-the-loop` — the operator stops being the wire

Work stops and waits for a person for two quite different reasons, and today they look identical from the outside.

In the first, the fleet has met something only a human can settle — an ambiguity in what was asked, a judgment call the
charge does not cover — and it is right to wait. What is wrong is *how* it waits: silently, on a board nobody is looking
at, until the operator next walks past a terminal. A chunk that stopped at nine in the evening has spent the night
achieving nothing, and the fleet's whole promise is the night. So a question goes and finds its person, arriving as a
message on a chat app they already have open and answering in a tap. The epic behind it builds the fan-out seam rather
than the bot: who subscribes to what, and how a chat identity resolves to a hub user, are decided once, so the next
channel is a binding rather than a rebuild.

In the second, nothing about the moment needs a human at all. CI has spoken, or a reviewer has left comments, and that
verdict is sitting somewhere the session that produced the work cannot see. A person reads it, understands it, and types
it back in — acting as a wire between two machines, adding nothing but latency. Routing those results back into the
owning session closes the loop without them, and the cap on how many rounds it may spend is the part that matters: an
agent arguing with CI indefinitely is worse than one that stops and asks, which returns the moment to the first kind
honestly.

The milestone is named for the distinction. A human belongs in the loop exactly where judgment lives, and nowhere else.

| Epic               | Slice | Status  |
| ------------------ | ----- | ------- |
| `epic:chat`        | full  | horizon |
| `epic:ci-feedback` | full  | horizon |

## `milestone:mobile` — the fleet in a pocket

The fleet works while its operator is away from the desk. That is the proposition, and it is also precisely when they
are least equipped to answer it: the surfaces that could unblock a chunk are a browser tab on a closed laptop and a
terminal in another room.

A phone changes that arithmetic completely. Not because anyone wants to run a fleet from a phone — nobody does — but
because the operator's part in a good night is small and interruptive. Read what the fleet is asking, decide, and put it
away. That is a phone-shaped job, and it is currently a laptop-shaped one.

The milestone holds three answers to that need and does not expect to spend all three. A progressive web app installs
onto a home screen cheaply and reaches one hub, which is the whole of what most operators run. Native apps on Android
and iOS earn something a wrapped page cannot: every hub the operator follows, and the runners beneath them, registered
together and read as one view — the way a Jira app registers its sites — with notifications carrying the reliability
only the platform's own can. Which of those blizzard invests in is deliberately undecided, and the two native epics
share a single registry and aggregation design so that whichever ships first settles it for the other.

All three stand on the notification fan-out that `milestone:human-in-the-loop` builds. Reaching a person is one problem,
and it is solved once.

| Epic           | Slice | Status  |
| -------------- | ----- | ------- |
| `epic:pwa`     | full  | horizon |
| `epic:android` | full  | horizon |
| `epic:ios`     | full  | horizon |

## `milestone:project-management` — work assembled, not pasted

Putting work in front of the fleet is currently an act of transcription. The operator knows which issues matter, and to
schedule them they leave the board, find each one in the forge, copy its id, come back, and hand them over one at a
time. For a single urgent fix that is nothing. For the twenty small items that make up a productive week it is the
friction that decides whether they get queued at all, and the product manager — whose entire relationship with the fleet
runs through the backlog and the board — feels it most, because scheduling is supposed to be one gesture and this is
twenty.

This milestone gives that act a surface of its own. The open items of every work source are browsable and searchable
from inside blizzard; the operator selects the ones they want, pulls them through ingest together, and shapes what lands
before it lands — many at once, grouped into a chunk where they belong together, split back apart where they do not, and
inspected side by side while deciding.

None of that makes blizzard a place where work is *defined*. The forge remains the backlog's home and the single owner
of what an item says; this is a surface for moving items into the fleet, not for authoring them. What it does ask of the
platform is real, though, and worth naming: browsing means the work-source seam gains an optional capability to
enumerate and search, a deliberate loosening of ingest-by-id that every future source binding will inherit.

It arrives after `milestone:projects` for a plain reason — a hub serving many projects that draw on many sources,
changes the shape of the screen enough that building it first would mean building it twice.

| Epic          | Slice | Status  |
| ------------- | ----- | ------- |
| `epic:intake` | full  | horizon |

## `milestone:future-state` — the fleet you can reach into

Everything before this milestone makes the fleet better at working unattended. This one is about the moments the
operator is present — and about the fleet still behaving sensibly in the long stretches when nobody is.

Presence is the weaker half of blizzard today. The fleet leases a whole environment for a chunk: every repository as a
worktree on one branch, a port band, provisioned resources, services actually running. That environment is the product's
strongest claim and no human can look at it. The operator judges a night's work from a diff, and
[`persona:product-owner`](./charter/personas/product-owner.md), whose entire question is whether the thing built is the
right thing, is asked to answer it from source. Opening the running application from the board turns that question back
into the one it actually is: use the feature and say. The cost is a genuine architectural problem rather than a screen —
the hub deliberately cannot reach into a runner, and preserving that is what keeps blizzard installable on a laptop
behind a router nobody controls.

The same absence shows up while work is moving. An operator watching a chunk drift has two levers, and both are bad: let
the night finish wrong, or take the session over and end its autonomy. What is missing is the thing a colleague does
across a desk — one sentence, said once, and the work carries on.

The unattended half is a question of rhythm. Blizzard has learned to watch itself — findings, trends, measurements taken
along named axes — and every one of those instruments waits for a person to start it. A fleet whose self-examination
only happens when its owner remembers to ask is not keeping its own house, and the gap widens exactly when the operator
is busiest. Scheduling is the small fix: a routine states how often it wants to run and how much breathing room it needs
between runs, the hub mints the work when that comes due, and the queue decides when it actually gets done. The
distinction matters — a fleet behind on real work should fall behind on its housekeeping first, not shoulder the
operator's queue aside to keep a calendar appointment. And once the fleet schedules its own work, not all of the queue
is the same kind: a production fix should not wait behind forty features, and the housekeeping the schedule mints should
take only the runners nobody else wants. Urgency lets a chunk say which, and the fleet reaches for the most urgent work
first, in the order the operator ranked it.

Some work no ranking can place well, because it cannot share the road at all. A rewrite from one language into another,
a framework swapped out from under the board, a dependency upgrade, a tech-debt sweep across the whole codebase — each
touches so much that any chunk running beside it lands on ground that moved underneath it, and the merge that reconciles
the two is exactly where functionality quietly goes missing. Today the operator who knows this can only stop the fleet
by hand and wait for it to drain. A chokepoint lets them say it once: the fleet finishes everything ranked ahead of the
chunk, passes through it alone, and spreads back out only once it clears.

Clearing the road is half of it; the other half is a lane built for what drives down it. Every development workflow the
fleet has today puts one agent at a time on a chunk, and the change behind `epic:architectural-sweep` reached its tens
of thousands of lines only because dozens of agents worked it in parallel, driven by hand from outside the fleet. An
`epic-dwf` graph brings that inside: epic-level development of twenty thousand lines or more, planned whole and built by
a swarm, and the broad work of thousands of small adjustments, each made by an agent looking only at its own corner —
landed as one change, through a chokepoint.

Rhythm has an inbound side too. Work reaches blizzard today because a person went and fetched it, which means the fleet
is idle through every hour that a person is asleep and something is going wrong in production. The systems that notice
trouble first already know how to make an HTTP request; letting them raise work into a project's own backlog closes the
last gap between an incident and a queued chunk without making blizzard the place either one is defined.

A swarm also needs to be watched while it runs. Behind one chunk's card, one session may be running dozens of agents,
and the board today shows only the card. An agent observer on the runner reads the session's transcripts and reports how
many agents are live, how much context each is carrying, and what they have spent, so the operator sees a swarm's vital
signs where they already look — and the fleet can finally see spend that happens inside a single node.

One more absence belongs here, and it is the first thing a stranger meets. Everything blizzard can do today it can only
show to someone who has already obtained a model key, a forge token, and a workspace, and who is willing to spend real
money to watch a chunk move. A fleet that runs against mock harnesses and a mock forge — deterministic, free, and wired
over the same seams the real one uses — lets a person configure a project, queue work, and watch it travel the graph
before committing anything. It serves the author of a workflow graph just as directly, who today learns what a graph
does by spending a night finding out.

A fleet that tunes itself needs a way to know whether a change helped. Today the harness engineer rewrites a reviewer's
prompt or moves a node to another model and judges the result from a few nights that felt better, nights that ran
different work with every node free to vary at once. A lab gives them a real experiment at the grain the question lives
in: it takes a graph the fleet already runs, keeps only the stretch under study, seeds everything upstream from a chunk
the fleet once ran, and stops each trial where `build` or `deliver` would have begun, so two configurations meet the
same input and differ in exactly one respect. The fleet's own history becomes its benchmark, and a proposal to change
how the fleet works can arrive carrying its evidence. The lab earns its place in the fleet the slow way first: run by
hand, as a winter lab outside blizzard, until a few experiments have changed a decision someone would otherwise have
made by feel.

The milestone closes with five pieces of ordinary platform maturity. Settings that every graph node restates want a name
to inherit instead, so an operator retunes the fleet's reviewers in one edit rather than nine. And the seams the mission
is built on want to be reachable from outside the wheel: interoperability is proven at two live bindings, and today
writing the second one means writing it in blizzard's own repository. A provider someone else can build, install, and
prove against a conformance suite is what turns a well-drawn seam into an actually open one. A hub's own configuration
wants to be declared where the rest of the estate already is: a Terraform provider lets the plan that builds the hub's
host also state its tenants, projects, sources, and repositories. And a hub shared by several tenants should narrate
each of them to that tenant's own observability backend, not only to the one its operator chose. Last, the board's own
suites earn the proof the backend's will already carry: mutation runs over the Angular workspace, waiting on a seam in
tooling blizzard does not own.

| Epic                       | Slice          | Status  |
| -------------------------- | -------------- | ------- |
| `epic:cadence`             | full           | horizon |
| `epic:steering`            | full           | horizon |
| `epic:signals`             | full           | horizon |
| `epic:preview`             | full           | horizon |
| `epic:worker-profiles`     | full           | horizon |
| `epic:provider-kit`        | full           | horizon |
| `epic:terraform-provider`  | full           | horizon |
| `epic:tenant-telemetry`    | full           | horizon |
| `epic:demo`                | full           | horizon |
| `epic:queue`               | priority       | retired |
| `epic:urgency`             | full           | horizon |
| `epic:chokepoint`          | chokepoint     | horizon |
| `epic:chokepoint`          | epic-dwf       | horizon |
| `epic:code-harness-window` | agent-observer | horizon |
| `epic:code-harness-window` | swarm-view     | horizon |
| `epic:mutation-testing`    | angular        | horizon |
| `epic:lab`                 | bench          | horizon |
| `epic:lab`                 | experiment     | horizon |
| `epic:lab`                 | lab-runner     | horizon |
| `epic:lab`                 | specimen       | horizon |
| `epic:lab`                 | verdict        | horizon |
| `epic:lab`                 | lab-board      | horizon |
| `epic:lab`                 | seeded-defects | horizon |
| `epic:lab`                 | evidence       | horizon |
