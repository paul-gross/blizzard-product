# Multica and blizzard, capability by capability

A two-way inventory: what Multica can do that blizzard cannot, what blizzard can do that Multica cannot, and the ground
the two products already share. Parent: [index.md](./index.md), which owns why Multica is worth a folder and how to read
a gap.

## The shape of each product

Multica is a Go server over Postgres with a Next.js web app, an Electron desktop app, an Expo mobile app, a CLI, and a
daemon that runs on any machine a team member connects. Its unit of work is an issue on a board, with the familiar
columns from backlog to done. Assigning an issue to an agent, mentioning one in a comment, or firing a schedule places a
task in a Postgres queue, and a daemon claims it, makes a directory for it, and starts the agent CLI there with a short
prompt that tells it to read the issue through the `multica` CLI. From then on the agent drives: it checks out the
repositories it needs, comments on the issue, moves the issue between columns, and opens the pull request itself. Its
durable state is the issue and its comments, the task rows, and a per-message execution log. It is about 680,000 lines
of production code with almost as much test code beside it, behind 619 migrations.

Blizzard is a Python wheel that ships two daemons, a CLI, and the compiled boards, over sqlite by default. It contains
no agent loop and no methodology. Two harnesses are live, Claude Code and OpenCode, and six packaged graphs move work
through stations whose prompts expect a methodology in the repository being driven. Its unit of work is a chunk, which
wraps one or more backlog items by reference. Its unit of isolation is a winter feature environment. Its durable state
is an append-only log of facts from which every status is derived.

The shortest way to hold the two in mind together: Multica is a task tracker that hired agents, and blizzard is a fleet
that refuses to be a task tracker. In Multica the agent holds the pen on its own record. In blizzard the record is held
somewhere the agent cannot reach. Nearly every row below follows from one of those two sentences.

## What Multica has that blizzard does not

### Every agent CLI the team already pays for

Multica's daemon detects the agent CLIs installed on its machine and offers each as a runtime: Claude Code, Codex,
Cursor, Copilot, OpenCode, Kimi, Qwen, Kiro, Grok, Devin, and sixteen more. One interface, a single `Execute` call that
takes a prompt and returns a stream of messages, is all a new harness must satisfy, which is why the list grew to
twenty-six in under a year. Users bring their own CLI, signed in with their own subscription, and Multica never touches
a model key.

Blizzard has two. [`initiative:adapters`](../../../epics.md) brings Codex next, and
[`epic:provider-kit`](../../../epics.md) would let someone else ship a binding, so the breadth is a schedule difference.
The asymmetry in depth is the design: a blizzard adapter must resume a session it did not start, return a verdict the
graph can route on, report usage, survive a compatibility contract pinned to a harness version, and hand a human the
session with one command. A Multica adapter must only run. Its popularity is still evidence that, for a team choosing a
tool, *does it run the CLI I already have* is the first question, and blizzard answers it for two names.

### The tracker itself

Multica is where the team's work lives. Issues, sub-issues, projects with their own leads and goals, custom statuses,
labels, an inbox of assignments and mentions, and parent issues whose children close in ordered stages make it a
project-management system in its own right, with agents filed alongside people on the same assignee list. A team that
adopts it replaces Linear or Jira rather than adding to them.

Blizzard's mission names the backlog a seam. A work source holds the items, the hub ingests specific ones by id, and the
board shows the fleet's work rather than the team's plan. That is a position difference, and the comparison confirms it.
It also raises a question the mission does not answer, taken up under
[What the differences ask](#what-the-differences-ask): a tracker that thousands of teams already use for agent work is a
candidate binding for the work-source seam.

### Columns that hand work along

A Multica project can carry a workflow: an ordered list of up to fifty status columns, each naming who handles it (an
agent, a squad, a member, the project lead) and up to eight thousand characters of instructions. Moving an issue into a
column reassigns it to that column's handler and starts their run with the step's brief as a handoff note. An agent
finishes its step by moving the issue on, or back when the work needs changes. Squads add a leader agent that is told to
delegate by mention rather than do the work. With both, a team can assemble a build column, an agent review column, and
a human approval column out of configuration alone.

This looks like a blizzard graph and is a different kind of thing, examined under
[A lifecycle the agent cannot skip](#a-lifecycle-the-agent-cannot-skip). As a surface, though, it is ahead: a team lead
shapes the flow from the board, in words, without writing YAML. The
[`persona:product-manager`](../../../charter/personas/product-manager.md) who never opens a repository can do it. A
graph editor on the board is not in the registry, and it is worth noticing that the most-adopted product in the field
made workflow authoring a non-engineer's job.

### A word to an agent mid-run

A comment on an issue whose agent is running is delivered into the current run rather than queued for the next. Multica
does this through each harness's native injection point, Claude Code's hooks, Codex's steer call, and the equivalents in
Grok, Kiro, and Cursor, gated by harness version, and records whether the message arrived before the run ended. On
harnesses without such a point, the comment wakes a follow-up run instead. [`epic:steering`](../../../epics.md) holds
the same promise for blizzard, so this is a schedule difference, and Multica's code is useful evidence that the
injection points exist natively in the harnesses blizzard means to support.

### A calendar and a doorbell

Multica's autopilots run on a cron schedule, from a webhook, or from an API call, and either file an issue or start a
run directly, with a policy for an autopilot that fires while its last run is still going. Wakeup rules on an issue can
fire on a time, an interval, another issue's change, or a pull request's state. Blizzard has no scheduler; routines run
when an operator runs them. [`epic:cadence`](../../../epics.md) and [`epic:signals`](../../../epics.md) hold the two
halves, so both are schedule differences. Two competitors in a row have shipped them first.

### Every surface, and the chat the team already has open

Beside the web board, Multica ships a desktop app that registers its own machine as a runtime, an iPhone and iPad app
built from source, and bots in Slack, Feishu, DingTalk, WeCom, and Telegram where a team member can mention an agent,
file an issue, or answer it without opening the board. A private chat window lets a person talk to one agent about
anything, with a project attached for context and no issue required. Blizzard has the board, a board complete from a
phone, the runner's panel, and the CLI. [`epic:chat`](../../../epics.md), [`epic:pwa`](../../../epics.md), and the
native apps hold the reach, so those are schedule differences. Free conversation with an agent outside a unit of work
remains a difference in kind, as it was against Fusion and Soleur.

### A team product with a price

Multica's workspaces carry owner, admin, and member roles, access scopes over which agents each member may run, and a
cloud tier sold per human seat, in which "agents never use a seat." Plan limits and prices live in a private cloud
service and are published nowhere; a self-hosted server simply runs without limits. Blizzard's hub is partitioned by
tenant with viewer and operator roles, and has no sign-up or billing, so the commercial surface is a position
difference. The per-seat price that leaves agents free is a pricing idea worth remembering, because it charges for the
people a product serves rather than for the fleet it runs.

## What blizzard has that Multica does not

### A lifecycle the agent cannot skip

Multica's lifecycle is enforced by asking. The agent's instructions say that `done` stays human, that a review column
means stop, that the next column is the one the step names. In code, an agent can set any status on any issue in its
workspace, including `done`, and can delete one. A project workflow refuses only a status it does not list, and refuses
an agent's move only when someone else has already moved the issue off its step; which column comes next is advice. The
task token an agent carries resolves to the role of the human who owns the daemon, so an agent on an admin's machine
reaches the workspace's admin routes, from member management to plugin installation, through the same middleware that
lets the admin in. Multica's guard against machine credentials covers billing, personal tokens, and a handful of other
routes, and its own comment says that any other actor "remains human-equivalent."

Blizzard governs by withholding the lever. Where a chunk stands, which gate it faces, and whether it lands are hub facts
a worker reaches only through the runner's verbs. A worker cannot advance itself, skip a gate, or merge, because none of
those is an action it holds, and the graph that decides them is immutable once minted. This was the deepest difference
against Soleur, and it is the deepest here. Multica's popularity shows that most teams do not yet ask for it. They will
ask the first time an agent marks its own work done.

### Delivery no model performs

Neither Multica's server nor its daemon ever pushes a branch or opens a pull request. The agent does both, with whatever
`gh` and git credentials sit in the daemon user's home, under an instruction not to wait for CI. Multica's integrations
with GitHub, GitLab, Gitea, and Forgejo receive webhooks and mirror pull-request and CI state onto the issue, and when
every linked pull request has merged they move the issue to done. Whether anything stops an agent merging its own work
is left to the forge's branch protection.

Blizzard's deliver node runs at the hub with no agent session near it, routes on the forge's merge state, heals a branch
that is merely behind without waking a model, refuses a pull request whose head carries commits it did not submit, and
consults a model only for a genuine conflict. Multica is ahead on the breadth of forges it can observe; blizzard's forge
seam has one binding, GitHub. It is behind on everything a forge can be asked to do.

### Claims that fence

Multica's queue is carefully built. A claim runs in one transaction that locks the agent's row, checks its concurrency
limit, and requires a recently heard runtime, and a sweeper re-delivers or fails work whose daemon went quiet. But a
completed task is fenced only by its status, and a retry is a new row, so a stale worker cannot overwrite its
successor's row and can still write to the issue. Its token is revoked when a task completes, fails, or is cancelled
through the normal paths, but not when the sweeper fails it, and a finished task is deliberately allowed to keep posting
comments. During a network partition, an agent that has been replaced keeps working on the issue under a live
credential.

Blizzard's leases are atomic and its submissions epoch-fenced, so a reaped worker that is still running cannot land
anything over the one that replaced it. That is [success criterion two](../../../charter/mission.md#success-criteria)
held as a structural guarantee, and it is the row in which a busy team would first feel the difference.

### Status that survives the machine

Multica's sweeper and the daemon's on-disk queue of unsent results recover a great deal: an unacknowledged dispatch is
sent again, a daemon that restarts reports its orphans, a run whose runtime vanished is failed after a grace period and
retried for a short list of causes. But recovery is a failed run and a fresh one. The issue moves back to `todo`, the
next run resumes the old session when it can, and the work in the directory survives because the directory does.
Blizzard derives every status from facts, so `kill -9`, a reboot, or a power cut at any instant costs at most the tokens
in flight, and the work resumes where it stood.

### An environment, not a directory

A Multica run gets an empty directory and checks out the repositories it needs into it as git worktrees from a cached
bare clone, side by side when it needs several. A later run on the same issue reuses the directory, and a garbage
collector removes it a day after the issue closes. Nothing provisions ports, services, or databases; the only per-run
resource is a slot number and a temporary directory. A blizzard worker is leased a whole winter feature environment,
reset to base and provisioned before it arrives, with every project repository as a worktree on one branch, an allocated
port band, provisioned resources, and services the agent brings up when the work needs them running.

### A budget the fleet obeys

Multica records tokens per run, provider, and model, and prices them in the browser from a static rate table. Nothing
stops a run, an agent, or a night for cost; the only limits it enforces are the cloud plan's issue count and autopilot
runs. Blizzard caps spend per chunk and over a rolling window, and its kill-switch stops the fleet rather than reporting
on it in the morning.

### Dependencies that hold

Multica has a table for issues that block one another, created in its first migration and read by nothing since; the
working ordering is the staged sub-issue, where a parent wakes when a stage of its children has closed. Since agents are
told to stop at review, a stage in practice waits for a human to mark each child done. Blizzard's queue will not let a
chunk be claimed until the chunk it stands on is done, from an edge the operator declares once.

### Cheap takeover

When a Multica agent stalls, a person can comment, steer, or reassign, and a follow-up run resumes the old session where
the harness allows it. Nothing hands the human the agent's session to work in themselves. Blizzard puts the human into
the stuck agent's full session context with one pasted command.

## Where the two products already agree

Both refuse to give an agent a credential it does not need. Multica mints a task token at claim time, bound to one task,
one agent, and one workspace, and refuses to start the agent rather than fall back to its own; blizzard starts workers
from an allowlisted environment that keeps the daemons' credentials out of the child. Both run every worker with
permissions bypassed: Multica on every harness it drives, with Codex in full-access mode as a stated product decision,
and blizzard by default. Neither ships an operating-system sandbox. Multica's security-model page says so in plain words
and tells its users to run the daemon as a dedicated user, in a container, or in a VM.

Both keep a durable, replayable record of every tool call a worker made and report usage per run and per model. Both
order the queue by priority and then age, with a concurrency limit per worker. Both let an agent ask a human and park
until the answer comes, Multica through a `blocked` status and an inbox and blizzard through questions at the hub. Both
pull context into the run rather than stuffing it into the prompt, Multica by having the agent read the issue through
its CLI and blizzard through the methodology in the repository. Both treat skills as files a harness loads, and neither
has agents that write their own memory. Multica's "turn a solved problem into a playbook" is a person writing a skill,
or importing one from a public registry, and the agents it runs forget as completely as blizzard's do.

Both are built by their own fleet, and both are public without being open source. Multica's repository describes itself
as open source; its licence is Apache 2.0 with conditions that forbid offering it as a hosted service to anyone outside
the licensee's organization, even free, forbid removing its branding from the interface, and let the producer relicense
contributions. That is source-available. Blizzard's repositories carry no licence at all, which leaves them all rights
reserved, a point the [Soleur comparison](../soleur/capability-comparison.md#where-the-two-products-already-agree)
already made. The fleets differ in one telling way. Multica's agent merges half its pull requests on a `main` branch
with no protection and no required checks, and the branch was green on 93% of the completed CI runs among its last
hundred pushes. Blizzard's `master`, gated by a strict queue, was green on 86% of the same measure.

The honest summary is that Multica has built the friendliest door into agent work the market has seen, and behind the
door the agents hold the keys.

## What the differences ask

Five decisions fall out of this comparison.

**The confinement posture now has a market baseline, and it is no posture.** The most-adopted product in the field runs
every harness with permissions bypassed and no sandbox, and its users have not objected. That weakens the case that
confinement is what wins a prospect, and leaves the case that [`milestone:polyglot`](../../../milestones.md) makes on
its own terms, that no worker should run with permissions dangerously bypassed, exactly where the
[Soleur comparison](../soleur/capability-comparison.md#what-the-differences-ask) left it: urgent and undecided. What
Multica adds is a model for the interim. Its security-model page states, in a few paragraphs, what a run can reach and
what an operator should do about it. Blizzard has no such page. Writing one is cheap, honest, and independent of the
larger decision.

**Breadth of harness is the first question a buyer asks.** Twenty-six against two is not a gap blizzard should close
adapter by adapter, because each of blizzard's adapters carries a contract Multica's do not. It is an argument for
[`epic:provider-kit`](../../../epics.md) arriving soon after Codex rather than long after, so that the long tail of CLIs
becomes someone else's adapter rather than blizzard's backlog. The order of those two in the registry deserves a look in
light of it.

**A popular tracker is a work source.** Multica has made itself the place where a great many teams already file work for
agents, and exposes all of it through an API and a CLI. The work-source seam exists so that blizzard can sit behind
whatever tracker a team already uses. A Multica binding, ingesting its issues and reporting the fleet's hold back onto
them the way the forge-status annotator does for GitHub, would put blizzard's guarantees behind Multica's door for the
team that outgrows asking agents to behave. Whether that is worth building is a decision. That it is the mission working
as written is not in question.

**Workflow authoring belongs to more than the engineer.** Multica lets a project lead shape a flow from the board in
plain words, and that is the feature its squads and columns are sold on. Blizzard's graphs are YAML that a harness
engineer writes and the hub mints. The graph's immutability and hub-held routing are the strength; the authoring surface
is a gap no epic covers. A board that edits and mints a graph, within the rules the hub already holds, is worth a
proposal.

**The landing's red rate deserves its second look.** The
[Soleur comparison](../soleur/capability-comparison.md#what-the-differences-ask) asked for one, and Multica sharpens the
question. Its agent merges unreviewed onto an unprotected branch and keeps that branch greener than blizzard keeps
`master` through a strict queue. Some of the gap will be what each suite tests, and how much of each suite runs on a
push. Some may be the merge skew the [field survey](../factory-field/index.md) named. The asymmetry is large enough to
settle before [`initiative:advanced-delivery`](../../../epics.md) designs its train.

The remaining differences are the design working. A tracker in its own right, an agent that drives its own record, and a
per-seat cloud are positions the mission already takes. Multica's customer, the small team hiring its first agents, will
find blizzard's strengths only after the first night an agent marks its own work done. It is worth being ready for that
night.
