# Milestones

What users will be able to do. A milestone is a destination stated in the user's terms, and it comes first: you declare
where the product must reach, then ask what work the journey requires — the milestone demands its epics, never the other
way around.

| Milestone                      | What users will be able to do                                                                                                                                                                                                                                                                                              |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `milestone:homeostasis`        | Run a fleet that holds its own pace and quality: what the fleet learns, spends, and builds is watched by the fleet itself, decay becomes filed work instead of quiet debt, work that stands on unfinished work waits its turn without a human holding it back — and the fleet begins to offer ideas of its own.            |
| `milestone:polyglot`           | Run the fleet on the coding harness of their choice — Claude Code, Codex, or OpenCode, first-class and mixable by node — with the safeties on: no worker runs with permissions dangerously bypassed.                                                                                                                       |
| `milestone:observability`      | Answer any question about their own fleet with the instruments they already trust: the numbers leave as files any warehouse or BI tool reads, and every night narrates itself as traces to whichever observability backend they run — no new dashboard to learn, and no waiting on blizzard to build the view they wanted. |
| `milestone:projects`           | Run every project from one fleet: a single hub hosting many projects' sources and queues, and a single runner per machine working all of them — a workspace per project, not a stack per project.                                                                                                                          |
| `milestone:hardening`          | Change how the fleet behaves without changing its code — every operational constant theirs to set — and find each new surface of the board behaving the way the last one did.                                                                                                                                              |
| `milestone:human-in-the-loop`  | Stop being the wire between the fleet and everything it needs: it reaches them wherever they are when a decision is genuinely theirs, and settles CI's verdict itself when it is not.                                                                                                                                      |
| `milestone:mobile`             | Carry the fleet in a pocket: watch the night, answer a question, and unblock a chunk from a phone, through notifications that arrive the way the phone's own do.                                                                                                                                                           |
| `milestone:project-management` | Assemble what the fleet works on from inside blizzard — browse the backlog, take many items at once, and shape them into chunks — instead of handing over ids one at a time.                                                                                                                                               |

## `milestone:homeostasis` — a fleet that keeps its own house

A fleet that ships every night can decay every night too, and the decay is quiet: a retrospective finding nobody turns
into work, a feature that cost three times what anyone guessed, a suite that stays green while it stops asserting, an
architecture bending one expedient commit at a time. Today the operator is the immune system — each of those signals
waits for a human to notice it, weigh it, and file something. The milestone is named for what it installs instead:
homeostasis, the way a living system holds its own vitals steady while the world outside changes. Pace and quality are
the fleet's vitals, and holding them level becomes the fleet's own job.

Through the charter's people: the harness engineer stops treating retrospectives as reading — a finding worth acting on
becomes a filed item with a stated priority, entering the same intake as any feature, and next week's fleet is
measurably different from last week's. The application architect's constraints hold while they sleep: fitness checks
name architectural drift while it is one commit old, and mutation runs prove the suites can fail, so a green gate means
behavior asserted rather than merely executed.

None of that learning scales past one machine unless its raw material travels. Today a worker's conversation lives and
dies as files on its own runner — workable for one runner, hopeless for twenty — so the milestone also centralizes the
fleet's conversations: every transcript, from every harness on every runner, flowing into one store the retrospective
sweeps read. A fleet can only learn from what it can reach.

And what the fleet can reach, it should count. One question asked of the transcripts by hand — how often are skills
used? — overturned an assumption: agents constantly, skills nearly never. Aggregated usage of the fleet's own machinery
— skills fired, agents spawned, context files read — turns the tending of the corpus from taste into evidence: the file
nobody reads gets reworded or removed on the numbers, not on a hunch. Counting is only half of it, though: a number that
takes a terminal session and a throwaway script to retrieve is learned once and then paid for again the next time
someone wonders. Making those numbers legible is a destination of its own, and it is reached in
`milestone:observability` rather than here.

Quality collects its guards here — fitness checks that name drift while it is one commit old, mutation runs that prove a
suite can fail — while pace has had none. Its costliest leak is work done twice: two chunks where the second stands on
the first, the first parked at a human gate, and an agent that reaches for the ground it expected, finds it missing, and
lays it again. The operator buys one idea twice and then pays a third time to reconcile the halves. Watching for that is
vigilance no person should have to supply, so the queue gains its first relation — an edge the operator declares once
and the queue honors from then on. A chunk that stands on another cannot be claimed until the one beneath it is done,
and comes ready on its own the moment it is; the fleet flows around it in the meantime rather than idling. Pace stops
depending on who happened to be watching the board.

The milestone also carries one deliberate maybe: ideation. A fleet that reads its own code every night is well placed to
notice what the product could become, not only what it should repair — so it proposes the occasional high-level feature
direction worth exploring, a recommendation to weigh rather than work it files. Whether those proposals earn their
reading is exactly what the exploration exists to find out.

| Epic                     | Slice        | Status    |
| ------------------------ | ------------ | --------- |
| `epic:self-sourced-work` | full         | delivered |
| `epic:garden`            | full         | delivered |
| `epic:transcripts`       | full         | delivered |
| `epic:analytics`         | full         | delivered |
| `epic:mutation-testing`  | full         | horizon   |
| `epic:cost`              | attribution  | retired   |
| `epic:queue`             | dependencies | delivered |
| `epic:ideation`          | full         | horizon   |

## `milestone:polyglot` — any harness, with the safeties on

Blizzard was built around a harness seam — spawn, resume, verdict — precisely so that no single coding agent would ever
be load-bearing. Today the seam has one occupant: every worker in every fleet is Claude Code, and every one of them runs
with the permission gate switched off. This milestone completes the harness story in both directions. Breadth: Codex and
OpenCode become first-class workers behind the same seam, chosen per fleet or mixed by node the way models already are.
Safety: the operator decides in advance what any worker may touch — whichever harness it runs — and the platform
enforces the decision rather than hoping. The name is what the fleet becomes: polyglot, fluent in more than one harness
and trusting none of them blindly.

Through the charter's people: the harness engineer runs the same chunk through two harnesses and compares the runs on
cost and quality, because harness choice has become a tunable rather than a fact of the platform. The application
architect dials trust by station — a build node's hands looser than a deliver-adjacent one's — and force-push is
structurally out of any worker's reach. And the operator stops lending the fleet their whole keyring: a worker holds the
least a chunk needs, and a worker gone wrong is contained by walls the operator chose, not by the agent's judgment on a
bad night.

| Epic            | Slice           | Status  |
| --------------- | --------------- | ------- |
| `epic:adapters` | breadth         | horizon |
| `epic:security` | worker-lockdown | horizon |

## `milestone:observability` — the fleet, in instruments you already trust

A fleet that works through the night produces a great deal of evidence about itself and hands almost none of it over.
The facts are all there — what each step cost and how long it took, what each gate decided, which files a worker
actually opened — and reaching any one of them costs a terminal session and a script written for that question and no
other. The reflex is to fix this by building somewhere to look. This milestone takes the opposite position: whoever runs
the fleet already has somewhere to look, and what they lack is any way to get blizzard's data into it.

That is the whole destination, and it is a deliberately modest one. Blizzard does not become an analytics product, does
not host a dashboard, and never learns the name of a single vendor. It grows two exits, shaped for the two kinds of
question people actually ask of a night's work.

The first is a record. The fleet's facts leave as files — one row per step, per event, per attempt, carrying names
rather than ids and nothing added up in advance — written wherever the operator asks for them. What receives them is not
blizzard's concern: a warehouse, a bucket, a laptop. The application architect who wants Monday to open on the week
against the week before builds that view once, in a tool they already know, and it goes on working without anyone
shipping them an endpoint for it.

The second is a narration. Each chunk's journey tells itself as it happens, in the protocol every observability backend
already speaks: the steps, the gates and what they decided, the waits in the queue, the deliveries the hub performs
itself. The harness engineer who suspects a gate rejects more than it should stops composing queries against a shape
nobody designed for the question, and follows the suspicion in a tool built for precisely that.

Neither exit substitutes for the other, because the questions differ in kind. One is about a moment — where six hours
went on the night something went wrong — and answers best as a trace that can be walked. The other is about a season —
whether the factory grows cheaper or dearer per thing shipped — and answers best as a table nobody has to reassemble. A
fleet that offers only the first can debug last Tuesday and say nothing about the quarter; one that offers only the
second can chart the quarter and lose every detail of the night that mattered.

What happens downstream of either exit belongs entirely to the person running the fleet: which backend, how long a year
is, whether anything is kept at all. Blizzard writes the files and sends the spans, and every decision after that is
someone else's to make.

| Epic               | Slice | Status  |
| ------------------ | ----- | ------- |
| `epic:fact-egress` | full  | horizon |
| `epic:tracing`     | full  | horizon |

## `milestone:projects` — one fleet, every project

Today the platform is single-project by silent assumption: one hub, and every source feeding it belongs to blizzard. The
operator who wants the same machinery working winter — or celestial frontier — stands up a second hub, a second runner,
a whole second stack on the same desk, and none of the stacks know of each other. This milestone makes *project* a
first-class idea: a grouping of both what to do and who does it. One hub hosts many projects, each with its own sources,
and a piece of work carries its project from ingest to landing.

The runner is rearchitected to match. A runner stops being the extension of a single workspace and becomes a host of
many: it configures a local workspace per project, and each workspace works for its project. Three projects on one
laptop means three workspaces and one runner — never three runners, and never three of everything above them. For the
operator, a desk full of stacks collapses into one: queue work against any project, watch all of it on one board, and
slice the view to a single project when only that one matters.

| Epic            | Slice  | Status  |
| --------------- | ------ | ------- |
| `epic:projects` | hub    | horizon |
| `epic:projects` | runner | horizon |

## `milestone:hardening` — the platform stops guessing on your behalf

A young platform is full of numbers somebody chose once. How long a lease lives, how many times a node retries before it
escalates, how long the reconciler sleeps between sweeps — each of them was settled during a build, by someone who was
not running this fleet, on this machine, against this repository. They are usually close enough to right. When one is
wrong it is wrong in code, and correcting it costs a change, a review, and a release.

The harness engineer feels that most sharply, because their whole craft is comparison: run the same chunk twice with one
variable moved and see what it cost. A constant they cannot reach is a variable they cannot move, and a question they
are simply unable to ask.

The board has the same problem in a different material. Every small surface it grows — a form, a picker, a confirmation
sheet — is invented, styled, and maintained by hand, and the count climbs with every epic that touches it. Hand-built
parts drift: two dialogs written six months apart agree on nothing, and the operator learns each of them separately.
Assembling the next one from a toolkit is faster to write, but the reason it matters is that it is quieter to use.

Hardening is what a platform does after it works. Nothing here adds a capability. Everything here makes the capabilities
already standing cheaper to change and steadier to meet.

| Epic              | Slice | Status  |
| ----------------- | ----- | ------- |
| `epic:config`     | full  | horizon |
| `epic:ui-toolkit` | full  | horizon |

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

It arrives after `milestone:projects` for a plain reason — a hub serving many projects, each with its own sources,
changes the shape of the screen enough that building it first would mean building it twice.

| Epic          | Slice | Status  |
| ------------- | ----- | ------- |
| `epic:intake` | full  | horizon |
