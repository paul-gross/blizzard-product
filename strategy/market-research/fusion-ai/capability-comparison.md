# Fusion and blizzard, capability by capability

A two-way inventory: what Fusion can do that blizzard cannot, what blizzard can do that Fusion cannot, and the ground
the two products already share. Parent: [index.md](./index.md), which owns how to read a gap.

## The shape of each product

Fusion is a TypeScript monorepo that ships an AI engine, a dashboard, a CLI, an Electron desktop app, and Capacitor
mobile apps. It contains its own agent loop: it calls model providers directly, holds the conversation, runs the tools,
and writes the plan. Coding CLIs are one execution option among several rather than the foundation. Its unit of work is
a task; its unit of isolation is a git worktree on a `fusion/{task-id}` branch; its durable store is PostgreSQL,
embedded by default and external when several nodes share a board.

Blizzard is a Python wheel that ships two daemons, a CLI, and the compiled boards, over sqlite by default. It contains
no agent loop and no model concept. Its unit of work is a chunk, which wraps one or more backlog items by reference; its
unit of isolation is a winter feature environment — worktrees across every project repository on a shared branch, with
its own ports, provisioned resources, and running services. Its durable state is an append-only log of facts from which
every status is derived.

Both statements are worth holding while reading what follows, because most of the differences below are consequences of
them rather than independent choices.

## What Fusion has that blizzard does not

### Provider breadth, and who gets to name a model

The premise to discard first is that blizzard has no model concept. It has one, and it sits where the mission would
predict. A packaged graph's session lineages declare an abstract tier and an effort — the advanced workflow gives
`planning` `model: ["blizzard:advanced"]` at `effort: high` and hands `code`, `gate`, and `plan-review` the basic tier —
alongside a compaction window and rotation thresholds fitted from a measured distribution of fleet sessions. Each runner
then maps `blizzard:advanced` to a native model id in its own configuration. A graph author who wants a cheaper reviewer
than builder says so today, and the graph stays vendor-agnostic while saying it.

What Fusion has that blizzard does not is breadth and immediacy. It speaks to Anthropic, OpenAI, Google, Ollama, Z.ai,
Kimi, a local llama.cpp server, and any user-defined OpenAI-, Anthropic-, or Google-compatible endpoint, because it owns
the agent loop and calls those providers itself. It splits choice into named lanes — executor, planner, validator,
merger, and two smaller ones — and resolves each through a four-level precedence that bottoms out in a per-task override
the operator can set from the New Task dialog, with per-task thinking levels and a fallback model that catches a
retryable provider failure.

So the honest difference is not presence but placement and reach. Blizzard binds the model twice — abstractly in the
graph, concretely in the runner — which is what keeps one graph driving twenty applications; the cost is that an
operator cannot retarget a single chunk at a different model without editing configuration, and the reachable set is
whatever the harness supports rather than whatever speaks HTTP. [`epic:adapters`](../../../epics.md) widens the harness
seam, which moves that ceiling without changing the shape.

### Iteration inside a node, and a planning artifact the platform owns

Both products run a plan-review-build loop, so the difference is where the loop lives rather than whether it exists, and
it is easy to state wrongly.

Blizzard ships the lifecycle loop. Its advanced graph is `plan` → `plan-review` → `build` → `verify` → `review` →
`pre-push` → `deliver`, with `resolve` and `retrospective` closing the cycles: the plan gate returns `must-fix` and the
work goes back to `plan`; a semantic conflict at `pre-push` sends the change back to re-earn its verification. Each
station is a fresh or resumed session of its own, and the two cold-eyes gates mint a new head every entry so a reviewer
never inherits the author's context. The lighter graphs collapse the same shape, and the basic harness lane carries an
explicit `iterate` node. There is nothing missing here, and describing Fusion's loop as one blizzard lacks would be
plainly wrong.

What Fusion adds is a second loop *underneath* that one. Its planning agent writes a `PROMPT.md` carrying steps, file
scope, and acceptance criteria, and the engine then walks that step list, running plan, review, execute, review around
each individual step, bounded by revision budgets and a replan cap that parks the task when exhausted. The platform owns
the decomposition of a feature into phases and drives the iteration through them.

Blizzard's [mission](../../../charter/mission.md) hands that layer down on purpose: which station the work is at is
blizzard's, and how a feature becomes phases inside a station belongs to the workflow tooling below it and to the
harness. The consequence is a distribution question rather than a capability one. Blizzard's `build` node arrives
expecting a methodology in the repository it drives, and where one exists the result is at least as good; where none
exists, Fusion arrives with its own in the box.

### Authoring a workflow by drawing it

Fusion's workflow editor is a dashboard surface. An operator inspects a built-in graph, duplicates it, and edits nodes,
columns, task fields, typed settings, model lanes, and review policy on a canvas, with a simplified list mode, lifecycle
warnings that offer to fix themselves, import and export, and a mobile layout. No engine fork is required to change how
work moves.

Blizzard's graphs are immutable YAML authored outside the product, and its board offers a graph explorer with retire and
re-enable rather than an editor. The underlying model is arguably the richer of the two — judgements, choices, gates,
and cycles, with migration when a chunk must move between graphs — but a harness engineer who wants to try a different
review lane edits a file and mints a graph where a Fusion operator drags a node.

### Conversation as a place the operator lives

Fusion's chat is not a notification channel. An operator talks directly to an agent on any model, asks a per-task agent
why a step failed, attaches files, answers a question card inline, and resumes a stream where it was left. Agents also
talk to each other: a built-in mailbox with inbox, outbox, and approvals carries delegation and hand-offs, and
experimental chat rooms let several agents coordinate with mention-routing and a cap on ambient participants.

Blizzard's human channel is precise and narrow by design — an ask, a gate, an escalation, a takeover — and there is no
way to simply speak with the agent holding your chunk short of entering its session. [`epic:chat`](../../../epics.md) is
a notification binding rather than a conversation surface, so the difference here is one of kind and not of schedule.

### Agents with names, roles, and a reporting line

A Fusion agent is a durable entity. It has a role, a place in a reports-to chain, a heartbeat on a schedule with
documented playbooks for what a manager or an IC does when it wakes, an auto-claim policy, and a share of the fleet's
tokens rendered on an org chart. Whole teams import at once: the product natively reads the `companies.sh` standard and
advertises 440-plus agents across sixteen prebuilt companies.

Blizzard's workers are anonymous. Runners are registered and named; the agent inside a lease is a session, and the fact
log records what it did rather than who it was. This is the one Fusion idea in this section that is neither scheduled in
the registry nor refused by the mission, and it is examined again under
[What the differences ask](#what-the-differences-ask).

### A backlog hierarchy inside the product

Missions decompose into milestones, slices, features, and tasks, with an autopilot that drives the tree, validation
contracts asserted at completion, fix-feature retries when a contract fails, and blocked-handoff semantics. Research
runs sit alongside them: a bounded investigation over web search, GitHub, and local documents, synthesized by a model
and convertible into tasks.

Blizzard refuses to be a task tracker; its equivalent of a mission tree is this repository, written by people. The
refusal is load-bearing — a second place where work is *defined* is exactly the drift the mission's non-goals guard
against — so the gap is a position rather than an omission.

### Sandboxing, where each product actually stands

This row belongs in neither inventory cleanly, and reading it as a Fusion advantage would be wrong.

Fusion confines an agent's shell through the platform's own facilities: bubblewrap on Linux, `sandbox-exec` on macOS,
each translating a declared policy into writable binds for the worktree and package store, read-only binds for the
system runtime, a tmpfs, an environment allowlist, optional network isolation, and a guard on the dashboard's port. The
coverage is uniform across execution paths, and the posture is permissive by default: the backend setting defaults to
`native`, sandboxing is marked experimental, and an unavailable backend either fails hard or falls back to unconfined
execution at the operator's choosing.

Blizzard confines through Landlock, in a fail-closed design that is the stronger of the two where it applies. The
OpenCode adapter binds the kernel syscalls directly and restricts itself before exec, so the boundary is inherited by
every descendant rather than depending on a helper binary; the worker is refused outright unless its workdir is a
disposable temporary directory; the system paths are read-and-execute only; and an unavailable or unenforceable boundary
raises rather than running the child unconfined. A second Landlock layer wraps the model's own shell tools and
deliberately withholds the runner's source tree from them.

The gap is coverage, not capability. That boundary is the OpenCode adapter's; the Claude Code path is governed by the
harness's own `--permission-mode`, which is a coarser instrument. So the accurate statement is that blizzard has built
the better confinement mechanism and has not yet spread it across every adapter, while Fusion has built the more uniform
one and leaves it off by default. The worker-lockdown slice of [`epic:security`](../../../epics.md) is about finishing
the spread and adding the network and credential dimensions, not about starting from nothing.

### Secrets, protocol servers, and inbound signals

Fusion carries an encrypted secrets store — AES-256-GCM at rest, scoped, with per-secret access policies wired through
to the tools an agent may call — and Model Context Protocol server configuration as a product surface, secret references
included. It also accepts HMAC-signed signals from Sentry, Datadog, PagerDuty, and generic webhooks, which is the
beginning of a fleet that reacts to production rather than only to a backlog.

### Grading the work after the fact

Fusion scores a finished task from zero to a hundred across agent performance, outcome quality, and process compliance,
blending a deterministic signal at seventy percent with a model's judgment at thirty, persisting the evidence behind
each score, and running the evaluation on a schedule. Blizzard records what happened with great care and renders no
grade.

### Memory that accumulates

Pluggable memory backends capture sessions and backfill per-chat transcripts, with opt-in vector recall; a nightly pass
distills insights into a written record; and the product claims agents that reflect on their own output and revise their
prompts as they learn a codebase. Blizzard's fact log is a record, not a memory an agent reads back.

### Surfaces, everywhere

Fusion ships an Electron desktop app for macOS, Windows, and Linux; native iOS and Android apps and an installable PWA;
a web dashboard; a terminal TUI; and a `fn` CLI covering tasks, projects, missions, settings, skills, and desktop
computer-use. Its Command Center renders token spend by model, productivity and human-hours-saved, duration percentiles,
language mix, an agent org chart, throughput timelines, and anomaly signals, across more than seventy themes and seven
README languages.

Blizzard has one board with a mobile glance shell. [`epic:pwa`](../../../epics.md), [`epic:android`](../../../epics.md),
[`epic:ios`](../../../epics.md), and [`epic:ui-toolkit`](../../../epics.md) all sit in the registry, so this is the
clearest case in the document of a schedule difference rather than a disagreement.

### A third-party plugin ecosystem

Fusion publishes a plugin SDK with a manifest, lifecycle hooks, HTTP routes, agent tools, runtime contributions, and
dashboard surfaces, plus a registry with installability metadata and a guide for authoring against a published CLI
without monorepo access. Shipped plugins already include Hermes, Paperclip, OpenClaw, a quality hub, reports, and a
bridge to Even Realities glasses. Blizzard's extensibility is at the seam, addressed to whoever writes a provider, not
to a contributor community.

### Ingest and scheduling ergonomics

GitHub import filters by label, optionally translates, attaches screenshots, and skips items it has already seen even
after an edit or a repository rename; PR and issue badges update live; a GitLab parity inventory and JIRA-derived branch
naming are both in the tree. Automations and routines fire on cron, webhook, or a manual trigger, at global or
per-project scope, with run history and signature verification.

Blizzard ingests a work item by id, and [`epic:intake`](../../../epics.md) holds the workbench that would make selecting
those items pleasant. Its routines exist but are gardening's, addressed at a scope and an axis rather than at arbitrary
work, and they carry no cron: a routine is run, not scheduled. The one half of this row blizzard already answers is the
return trip — a periodic best-effort sweep reflects a chunk's derived status onto its forge items as labels, writing
only the diff and holding no state of its own, so a mid-sweep crash self-heals.

## What blizzard has that Fusion does not

### An environment, not a checkout

This is the difference that matters most, and Fusion's own documentation draws the line clearly. A Fusion workspace
project is a non-git parent directory whose direct children are repositories; a task acquires one worktree per
configured member under a task directory, and each fresh member runs a bounded, root-level dependency bootstrap across a
matrix of recognized package managers. That is where it stops. There are no ports, no services, no provisioning hooks,
and no environment lifecycle.

A blizzard worker is leased a whole winter feature environment: every project repository as a worktree on one branch, an
allocated port band, provisioned resources, and services actually running. An agent that must exercise a system end to
end — drive the API it just changed, click through the UI it just built, watch the two talk to each other — can do that
in blizzard.

Fusion reaches for the same thing from the other side and gets part of the way. Its Quality plugin can start a
task-scoped preview server against a task's worktree on a free port, so an operator can exercise one change in a
browser, and its browser-verification step is a workflow-declared optional gate. What is absent is the composition: one
supervised process for one task's checkout, spawned on demand, is not a port band, a provisioning lifecycle, and a set
of interdependent services that come up together and tear down together. The distinction matters most for exactly the
work blizzard's mission is built around — a chunk spanning four repositories whose services must talk to each other —
and it is fair to call this a category Fusion has entered at its edge rather than one it has not entered at all.

### Status that cannot be wrong

Blizzard stores nothing observable as a flag. A chunk's status, a runner's liveness, and every brake derive from an
append-only fact log, which is what lets the product promise that `kill -9`, a reboot, or a power cut at any instant
costs at most the tokens in flight.

Fusion stores status and repairs it. Its run-audit catalogue is a long, carefully maintained list of reconciliation
events — orphaned pending step results rewritten to failed, unproven review approvals reopened, phantom executor
bindings reclaimed, stale duplicate decisions cleared, engine-downtime timing anchors shifted — because the board can be
wrong and a sweep must notice. The engineering behind those sweeps is serious and the documentation is better than most,
so this is not a quality judgment. It is a difference in kind: one product recovers from incorrect state and the other
cannot hold incorrect state.

### Delivery the platform executes, and cannot execute twice

Blizzard's deliver node runs at the hub, with no agent session anywhere near it, and exactly-once landing is structural:
atomic claims plus monotonic lease epochs fence a reaped-but-still-running worker out of the merge queue rather than
hoping a retry sorts it out.

Fusion holds the closer half of this than any other row — it has central task claims and lease epochs too — but its
landing runs on the node that owns the task, with a *merger model lane*, which is to say a language model with a hand on
conflict resolution. Blizzard's shipped PR-and-CI delivery policy makes the contrast concrete: it opens a pull request
per repository and routes on the forge's own `mergeable_state`, merging what is clean and self-healing what is merely
behind without waking a model at all, so the model is consulted only for the genuinely dirty conflict. And its
multi-repository landing is, in its own words, non-atomic: repositories land in a deterministic loop, an earlier one can
succeed while a later one fails, a `landedSha` proof exists to stop a retry from squashing twice, and a member whose
branch has vanished without proof parks the task for manual intervention.

### Graphs that hold still

Every edit to a blizzard graph mints a new one, so anything pinned to a graph can trust it forever, and a chunk changes
graphs only through an explicit migration. Fusion's workflows are edited in place. The editable graph is the friendlier
surface and the immutable graph is the one an audit can rest on; which is preferable depends entirely on whether anyone
will later ask what rules a particular chunk actually ran under.

### No model of the application, on purpose

Blizzard holds no per-application configuration at all — no convention registry, no per-app graph variants, no place to
declare how a project is tested — on the reasoning that a competent agent dropped into a capable workspace will find
those answers in the repository, where they stay correct as toolchains change. One graph drives twenty unrelated
applications unchanged.

Fusion's settings reference runs to roughly two thousand lines, its dependency bootstrap enumerates package managers by
name, and its Plan Review can fail a task with "Dependencies are not installed." That is a real product surface serving
real operators, and it is also the mechanism by which an orchestrator slowly acquires an opinion about each application
it drives. The two products have taken opposite bets here, and this is the one worth watching over the next year.

### Work shapes that are not one-to-one

A blizzard chunk wraps one *or more* backlog items, may span several feature environments and several repositories at
once, and may fan out to subagents inside one lease. Fusion's task is singular; workspace mode gives it several
repositories, but membership is a project-level configuration every task inherits rather than a shape chosen per unit of
work, and there is no environment cardinality to vary.

### Gardening

Blizzard turns the fleet on its own codebase along named axes, addressed at scopes, run by routines that record a
measurement each time so a scope acquires a trend. A finding is anchored to a file, a line, and the rule it violates,
and carries its own fact history — first observed, last seen, live or gone — rather than a stored flag, closing through
explicit verbs. A garden proposal then answers one or more findings with an argument and the evidence attached, and the
operator passes or accepts it on that argument.

Fusion has audit documents written by hand and an evals subsystem that grades finished tasks. It has no standing
mechanism that converts a codebase into tracked, closable, trending findings, and nothing that proposes a sweeping
change and defends it.

### Stepping into the session

When blizzard escalates, one pasted command drops the operator into the stuck agent's own session with its full context
intact. Fusion offers steering, chat, and a task terminal — friendlier in daily use, and genuinely better for a nudge —
but not entry into the agent's session as it stands.

### A human-entry taxonomy small enough to hold in the head

Blizzard has four ways a person enters the loop and two parked conditions: `waiting_on_human`, which means input was
invited and stops the reap clock, and `needs_human`, which means the system is out of moves. Both derive from open
facts. Fusion's equivalent is spread across oversight levels, awaiting-approval reasons, pause reasons, and a
notification event list, each individually sensible and collectively a larger thing to learn.

### A trust boundary that lets a runner hide

The blizzard hub never reaches into a developer's machine. All contact is runner-initiated, and operator controls are
declarative state a runner reads on its own next tick, which is what lets a runner sit behind NAT on a laptop with
nothing inbound. Fusion's nodes share a Postgres and exchange peer HTTP for membership, health, and optional credential
fan-out. The permissive topology buys reach; the restrictive one buys deployability on machines nobody wants to expose.

### Spend that stops itself

Fusion meters tokens well and visualizes cost better than blizzard does, and its per-task budgets alert on a soft cap
and pause on a hard one, with per-size tiers. Blizzard denominates its limits in dollars — a per-chunk cap that parks
the chunk, and a runner ceiling that engages that runner's local pause brake — which is the form an operator deciding
what a night may cost actually thinks in.

### One artifact to install

Blizzard is a single wheel carrying both daemons, the CLI, and the compiled boards, with sqlite by default and Postgres
as one configuration key, and a daemon that refuses to start on a store-revision mismatch rather than migrating silently
underneath the operator. Fusion is a pnpm monorepo with embedded Postgres, Electron, and Capacitor targets. The trade is
breadth of surface against weight of installation.

## Where the two products already agree

Several things that look like differentiators are, on inspection, shared ground, and treating them as differentiators in
a conversation with a prospective user will not survive contact.

Both run multiple nodes against one shared board, and both enforce exclusive execution through a central claim plus a
lease epoch. Both let one unit of work touch several repositories. Both move work through a graph of nodes with gates
and cycles, and — the row most easily got wrong — both run a plan, review, build, review loop with a cold reviewer and a
route back when the gate refuses. Both treat human involvement as a dial rather than a doctrine — Fusion through an
oversight level per workflow or task plus gate nodes, blizzard through gate nodes plus a runner's ability to impose one
by node name. Both meter tokens and cost and can stop a runaway. Both ingest GitHub issues and deliver either by merging
or by opening a pull request. Both are open source.

The honest summary is that Fusion has built more of the surface and blizzard has built more of the floor.

## What the differences ask

Four decisions fall out of this comparison, and only one of them is urgent.

**Worker lockdown is uneven rather than absent.** The Landlock boundary blizzard already enforces is the better
mechanism of the two — inherited by descendants, fail-closed, refusing a workdir outside a disposable directory — but it
governs the OpenCode adapter, while the Claude Code path relies on the harness's own permission mode. Fusion's coverage
is uniform and its default is off. The argument for promoting the worker-lockdown slice of
[`epic:security`](../../../epics.md) therefore rests on spreading a boundary that already exists to every adapter, and
on the network and credential dimensions neither product has finished, rather than on a bare gap.

**Surface breadth changes the schedule, not the thesis.** Phones, chat, an intake workbench, and a component toolkit are
all already in the registry. Fusion having them first is a reminder of what an operator notices in the first ten
minutes, not evidence that the floor was the wrong place to start.

**Several gaps are the design working.** The mission hierarchy, the platform-owned planning artifact, the loop
underneath the node, the model lanes, and the per-application configuration surface are each either a stated non-goal or
a layer blizzard deliberately leaves to the workspace and the harness. Reading them as backlog would be reading the
comparison backwards. The one caution attached to this row is distribution rather than design: a lane blizzard delegates
still has to be there in the repository being driven, and a prospect who arrives without one will experience the
delegation as an absence.

**Agent identity deserves a deliberate answer.** Named, durable agents with roles and a reporting line are the one
Fusion idea in this document that the mission does not refuse and the registry does not already hold. They change what
an operator can ask — which reviewer is best, which worker keeps escalating, who should take the next chunk of this kind
— without touching the fact log, the graph, or the lease protocol. The answer may well be no. It should be a considered
no.
