# Milestones

What users will be able to do. A milestone is a destination stated in the user's terms, and it comes first: you declare
where the product must reach, then ask what work the journey requires — the milestone demands its epics, never the other
way around.

| Milestone               | What users will be able to do                                                                                                                                                                                                                                                                                   |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `milestone:homeostasis` | Run a fleet that holds its own pace and quality: what the fleet learns, spends, and builds is watched by the fleet itself, decay becomes filed work instead of quiet debt, work that stands on unfinished work waits its turn without a human holding it back — and the fleet begins to offer ideas of its own. |
| `milestone:polyglot`    | Run the fleet on the coding harness of their choice — Claude Code, Codex, or OpenCode, first-class and mixable by node — with the safeties on: no worker runs with permissions dangerously bypassed.                                                                                                            |
| `milestone:projects`    | Run every project from one fleet: a single hub hosting many projects' sources and queues, and a single runner per machine working all of them — a workspace per project, not a stack per project.                                                                                                               |

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
behavior asserted rather than merely executed. And the operator learns what a feature cost, where before only a night
had a price — spend rolled up to the work item that caused it, trending across weeks, so the factory's unit economics
are a fact on the board rather than a feeling.

None of that learning scales past one machine unless its raw material travels. Today a worker's conversation lives and
dies as files on its own runner — workable for one runner, hopeless for twenty — so the milestone also centralizes the
fleet's conversations: every transcript, from every harness on every runner, flowing into one store the retrospective
sweeps read. A fleet can only learn from what it can reach.

And what the fleet can reach, it should count. One question asked of the transcripts by hand — how often are skills
used? — overturned an assumption: agents constantly, skills nearly never. Aggregated usage of the fleet's own machinery
— skills fired, agents spawned, context files read — turns the tending of the corpus from taste into evidence: the file
nobody reads gets reworded or removed on the numbers, not on a hunch. Counting is only half of it, though: a number that
takes a terminal session and a throwaway script to retrieve is learned once and then paid for again the next time
someone wonders. So the evidence surfaces where the operator already looks — a page that opens on the week against the
week before, and gives way to whatever dimension the question turns out to need.

Quality collects its guards here — fitness checks that name drift while it is one commit old, mutation runs that prove a
suite can fail — while pace has had none. Its costliest leak is work done twice: two chunks where the second stands on
the first, the first parked at a human gate, and an agent that reaches for the ground it expected, finds it missing, and
lays it again. The operator buys one idea twice and then pays a third time to reconcile the halves. Watching for that is
vigilance no person should have to supply, so the queue gains its first relation — an edge the operator declares once
and the queue honors from then on. A chunk that stands on another cannot be claimed until the one beneath it is done,
and comes ready on its own the moment it is; the fleet flows around it in the meantime rather than idling. Pace stops
depending on who happened to be watching the board.

The milestone also carries one deliberate maybe: ideation. A fleet that reads its own code every night is well placed to
notice what the product could become, not only what it should repair — so it offers the operator an occasional brief of
high-level feature directions worth exploring, recommendations to weigh rather than work it files. Whether those briefs
earn their reading is exactly what the exploration exists to find out.

| Epic                     | Slice        | Status    |
| ------------------------ | ------------ | --------- |
| `epic:self-sourced-work` | full         | delivered |
| `epic:garden`            | full         | delivered |
| `epic:transcripts`       | full         | delivered |
| `epic:analytics`         | full         | delivered |
| `epic:visual-analytics`  | full         | horizon   |
| `epic:mutation-testing`  | full         | horizon   |
| `epic:cost`              | attribution  | horizon   |
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
