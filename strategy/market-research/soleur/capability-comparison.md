# Soleur and blizzard, capability by capability

A two-way inventory: what Soleur can do that blizzard cannot, what blizzard can do that Soleur cannot, and the ground
the two products already share. Parent: [index.md](./index.md), which owns why Soleur is worth a folder and how to read
a gap.

## The shape of each product

Soleur comes in two layers. The first is a plugin for coding harnesses: sixty-seven agents, a hundred and three skills,
three commands, and a handful of hooks, written primarily for Claude Code and carried to Codex and Devin by translation
tables that map one harness's tools onto another's, with Grok mirrored and Cursor stubbed. It contains no orchestrator.
Each step of its lifecycle is one model session following long markdown instructions; the review skill alone runs to
about 120,000 tokens. A session starts in one of three ways: a founder types into a live harness, a cron function spawns
`claude --print`, or a GitHub Actions job runs the harness against the repository. The second layer is a hosted web app:
a Next.js front end over a custom Node server that runs Claude Code in-process through the Agent SDK, inside one
multi-tenant container on one Hetzner host, over Supabase Postgres, with a self-hosted Inngest for queueing and cron.
The unit of work is a conversation or a session. The unit of isolation is a git worktree, or a clone of the founder's
repository on the host. The durable state is a markdown knowledge base committed to the founder's own repository, plus
Postgres rows for conversations and engine runs.

Blizzard is a Python wheel that ships two daemons, a CLI, and the compiled boards, over sqlite by default. It contains
no agent loop and no methodology. Two harnesses are live, Claude Code and OpenCode, and six packaged graphs move work
through stations whose prompts arrive expecting a methodology in the repository being driven. Its unit of work is a
chunk, which wraps one or more backlog items by reference. Its unit of isolation is a winter feature environment. Its
durable state is an append-only log of facts from which every status is derived.

The shortest way to hold the two in mind together: Soleur is a methodology that grew a scheduler, and blizzard is a
scheduler that refuses to own a methodology. Nearly every row below follows from that.

## What Soleur has that blizzard does not

### A methodology in the box

Soleur's engineering loop is brainstorm, plan, work, review, compound, with a router command that classifies what the
founder wants and a `one-shot` skill that chains the whole sequence from worktree to merged pull request. Planning runs
in a subagent whose context is thrown away afterward. Review is a panel of separate subagents, each starting from fresh
context: eight sit on every code change, chosen from seventeen, with more seated by the kind of change. The plan has its
own review panel, and the ship step stacks eighteen named gates into one phase before it pushes. All of it carries from
one harness to another as prompt text, which is how a Devin or Codex user gets the same lifecycle as a Claude Code user.

Blizzard's equivalent exists, and it is not blizzard. The planning, cold review, and verification that blizzard's
stations invoke live in this workspace's winter-workflow and blizzard-context extensions: about sixteen skills and
fifteen agents, all engineering, installed by the workspace rather than shipped in the wheel. That is the
[mission's](../../../charter/mission.md) placement working as written, so this is a position difference. It does sharpen
the caution the Fusion comparison raised: a prospect who arrives without a lane experiences the delegation as an
absence, and Soleur shows that the lane can be a product in its own right.

It also shows what the lane costs when nothing above it holds the shape. Soleur's skill markdown totals 4.6 MB, much of
it incident-specific warnings that its learning loop wrote back into the skills, and the byte ratchet it adopted only
stops the growth. A methodology that must also do the fleet's job has to carry the fleet's rules in its prompts.

### Memory that writes itself back

Every Soleur session that finishes runs a compound step. It writes a dated learning, cross-references it, and appends a
one-line warning directly into whichever skill or agent file was active. Learnings arrive at about 117 a week and number
some 2,700. A weekly job has a model cluster them and propose edits to the always-loaded rules as draft pull requests
that never merge themselves. Read-back is partly automatic: the full rule file is injected at every session start,
planning spawns a researcher that searches past learnings, and the skills, having absorbed their own warnings, are read
every time they load.

Blizzard records with care and remembers nothing on its own. A chunk's retrospective is an asset attached to that chunk.
Review findings land in the hub's findings store, and gardening closes a loop on the codebase rather than on any agent's
recall. The closest analogue to compounding is this workspace's retrospective analysis, which clusters recurring
complaints across closed chunks and drafts a fix for a rule, a prompt, or the registry, then waits for a human to
approve it. No epic covers agent memory, and the mission does not refuse it. This is a genuine gap, and the evidence
Soleur supplies about it is unusually candid. Its own benchmark found the knowledge-base search skill recalling worse
than plain grep, 0.77 against 0.95 on exact-phrase queries, and the write-back loop is what swelled its skills to their
present size. The part of Soleur's design that holds up best is the human-gated promotion. It is examined again under
[What the differences ask](#what-the-differences-ask).

### A fleet that keeps its own calendar

Soleur's hosted side carries sixty-one cron functions, about eighteen of which start an unattended harness session. Each
one starts from a fresh shallow clone, with an allowlisted environment, a GitHub App installation token rather than a
personal one, a fifty-minute hard stop, a turn cap, and a dollar cap calibrated from that job's measured cost. A daily
bug-fixer takes the lowest-priority open issue, attempts a single-file fix, and merges it without a human only when the
author is the bot, the diff touches one file, and the issue was low priority. If the default branch then goes red, a
monitor reverts the commit. A weekly drain takes every issue under a label, groups them by code area, picks the largest
cluster, and hands it to `one-shot`. Much of this calendar is Soleur running its own company, posting a daily community
digest, auditing its own SEO, and drafting its own content, rather than anything a customer would schedule.

Blizzard has no cron anywhere. Routines run when an operator runs them, and its Claude Code workers are explicitly
denied the harness's scheduling tools. [`epic:cadence`](../../../epics.md) holds the hub minting a routine's work when
its interval comes due, so the calendar is a schedule difference. The label drain splits in two. Browsing and bulk
ingest are [`epic:intake`](../../../epics.md)'s, also scheduled. A machine choosing which issues belong together in one
run is a position: blizzard executes the shape it is handed and never computes it.

### Work that starts from an event, as a draft

When something happens on a founder's repository, a review waiting, CI failing, a high-priority issue opened, a
vulnerability advisory, Soleur's hosted app has an agent draft a response and places it as a card in a *Today* queue.
The founder sends it, edits it, or discards it, and only sending starts the agent's turn.
[`epic:signals`](../../../epics.md) holds inbound events raising work, so the trigger is scheduled. The draft card is
the more interesting half: a proposal sized to approve with one tap, which is the shape the product owner persona
answering from a phone is waiting for.

### Conversation, push, and a phone

The hosted app is a conversation before it is anything else. A founder chats with one of eight department leads over a
WebSocket that resumes a session or replays the stream after a dropped connection, answers questions inline, and
approves gated actions, with web push and then email reaching them when the tab is closed, all inside an installable
progressive web app. Blizzard has a mobile glance board and no notifications or chat of any kind.
[`epic:chat`](../../../epics.md) and [`epic:pwa`](../../../epics.md) hold the notification and phone halves, so those
are schedule differences. Speaking freely with the agent that holds a chunk remains a difference of kind, as it was
against Fusion.

### Trust dialed per action, not per station

Soleur classifies what an agent might do, such as answering a failed CI run, posting a public thread, or bumping a
dependency, and lets the founder set each class to act on its own, to draft for one-click approval, to ask every time,
or to act and report in a digest. Money, legal commitments, and credentials are excluded by construction. Blizzard dials
trust per station: a gate node in the graph, or a gate a runner imposes by node name. Per-station is the right grain for
a lifecycle, and the triage router already sends different kinds of work down different graphs. A per-class dial is a
finer instrument for the same [HITL-to-HOTL transition](../../../charter/mission.md) the mission promises. It is worth
remembering, though nothing here makes it urgent.

### A company, not only a codebase

The finance, legal, marketing, sales, operations, and support agents are prompts: department leads that assess and
delegate, holding no tools of their own. Real reach lives in skills and jobs, among them Stripe invoicing in test mode,
posting to Discord, X, Bluesky, and LinkedIn, provisioning on Hetzner and Cloudflare, feature flags in Flagsmith, and a
GDPR gate with its own scripts. The mission's subject is a fleet of *coding* agents. Blizzard's architecture admits work
that is not software, but no graph, persona, or epic serves a business function, so this is a position difference, and
the comparison confirms it.

### A hosted product with tenants and a price

Soleur's hosted app has sign-up, organizations, workspaces with members and roles, row-level security on fifty-six
tables, per-founder database tokens so the server holds no signing key, customer-supplied model keys encrypted under
keys derived per user, and Stripe plans at $49, $149, and $499 a month that cap concurrent conversations at two, five,
and fifty. The tenancy is thinner at runtime than in the data. Every tenant's agent shares one container on one host,
separated by a sandbox rather than a machine, and the server refuses to run as more than one replica because live
sessions, locks, and approval gates are held in process memory. Blizzard's hosted hub is its author's own fleet, and
[`epic:multi-tenancy`](../../../epics.md) partitions a hub without sign-up or billing, so the commercial surface is a
position difference.

### Confinement, where Soleur's hosted path is ahead

This is the one row in which Soleur is ahead on the floor rather than the surface.

Soleur's hosted agents run inside bubblewrap that fails closed when unavailable. They are denied reads across every
other tenant's root, may write only to their own workspace, have no network egress beyond an allowlist, and sit under a
seccomp filter and an AppArmor profile, with a canary proving the sandbox on every deploy. Its unattended cron sessions
add an egress allowlist at the host firewall. Its plugin layer has none of this. There, permission settings and denied
read paths are the whole boundary, and its own hook code says outright that hooks living in the operator's checkout,
editable by the agent they constrain, are no enforcement boundary.

Blizzard ships no operating-system confinement for a live worker on either harness. One autonomy setting maps onto each
harness's own permission system, and it defaults to `dangerous`, which on Claude Code is `bypassPermissions`. What does
hold under every value is real but narrower: an allowlisted environment that keeps daemon credentials out of the child,
the runner's own tool denials, and a process group that dies with its parent.

The Fusion comparison credits blizzard with a fail-closed Landlock boundary on the OpenCode adapter. That boundary
exists, but it confines only the OpenCode compatibility diagnostic. It refuses any workdir outside a temporary
directory, so a feature environment can never run under it, and the live adapter never uses it. That was already true
when the Fusion comparison was written, so the row there is wrong rather than stale.
[`milestone:polyglot`](../../../milestones.md) promises that no worker runs with permissions dangerously bypassed, and
today's default does exactly that.

## What blizzard has that Soleur does not

### A layer above the session

Soleur keeps a session moving by telling it to. Its pipeline skills carry blocks that instruct the model not to end its
turn, and a stop hook borrowed from the Ralph loop catches a session that tries. Concurrency in the plugin is a set of
advisory file leases that expire on their own window, and serialization is GitHub's merge queue. The hosted app does
better. It keeps heartbeated concurrency slots and a fenced per-worktree write lease with a generation counter, but
those are leases on conversations, not claims on units of work in a queue.

Blizzard is that missing layer: a queue with promotion and dependencies, atomic claims, epoch-fenced completion, and a
reconciliation loop on every runner that reaps, pulls, fills, and advances, across any number of runners on any number
of machines, each reaching the hub from behind its own NAT. Soleur's calendar, its label drain, and its single-file
bug-fixer are each a small, careful answer to a question blizzard answers once for every chunk.

### Status that survives the machine

When Soleur's hosted container restarts, every live session is dropped. On boot, a sweep marks any conversation that was
active more than five minutes earlier as failed, and uncommitted work survives only because it is snapshotted to
checkpoint refs in git. Running on more than one host is a decision record marked *adopting*. Blizzard derives every
status from facts, so `kill -9`, a reboot, or a power cut at any instant costs at most the tokens in flight, and the
work resumes where it stood.

### An environment, not a clone

A Soleur worktree is a checkout with the main repository's `.env` files copied into it, and a hosted workspace is a
clone in a directory. Nothing provisions ports, services, or databases for a task. A blizzard worker is leased a whole
winter feature environment, reset to base and provisioned before the worker arrives: every project repository as a
worktree on one branch, an allocated port band, provisioned resources, and services the agent brings up when the work
needs them running. One chunk may span several repositories, where Soleur's world is one founder's monorepo.

### Delivery no model performs

Soleur's ship step is a skill of nearly three thousand lines that the model executes, gate by gate, before it asks
GitHub to auto-merge and then polls the release. The mechanical half is strong: twenty-three required checks managed as
code, a merge queue that lands one entry at a time only when everything is green, and no human approval required. The
judgment half is a model working through a checklist. Blizzard's deliver node runs at the hub with no agent session near
it, routes on the forge's own merge state, heals a branch that is merely behind without waking a model, refuses a pull
request whose head carries commits it did not submit, and consults a model only for a genuine conflict.

### Rules the governed cannot reach

Soleur governs its agents largely by telling them. Ninety-eight rules are injected into every session, about half
enforced by nothing but the model's attention, and its sixty-odd guarding hooks live in the founder's own repository,
not in the plugin it ships. Its own commentary puts the history plainly: every hook exists because a rule failed. The
only layer an agent cannot edit is CI and the branch ruleset.

Blizzard governs by withholding the lever. Where a chunk stands, which gate it faces, and whether it lands are hub facts
that a worker reaches only through the runner's verbs. A worker cannot advance itself, skip a gate, or merge, because
none of those is an action it holds. The graph that decides them is immutable once minted, so a later question about
which rules a chunk ran under has one answer. This is the deepest difference in the document. A hundred and forty-nine
post-mortems in eight months are what a fleet records when its rules can only be stated.

### Cheap takeover

When a Soleur session stalls, the founder resumes it through the SDK or, failing that, through a replay of its history
as a new prompt. Blizzard puts the human into the stuck agent's full session context with one pasted command.

## Where the two products already agree

Both refuse to be a task tracker. Soleur's hosted app has no task table at all; its board is the founder's GitHub
issues, read live. Both run a plan, build, review, land loop with reviewers who start from fresh context, and both carry
findings forward, Soleur by filing them as issues and blizzard by recording them in the hub. Both bound spend in
dollars, per unit of work and over a rolling window, and both stop rather than explain. Both start child processes from
an allowlisted environment instead of the daemon's own, both keep secrets encrypted at rest, and both put Claude Code
first with a second harness live beside it, Devin for Soleur and OpenCode for blizzard. Both leave Codex unfinished:
Soleur's engine is coded and switched off, and blizzard's is [`epic:adapters`](../../../epics.md). Both are built by one
person and a fleet, dogfooded on themselves, and deployed from their own main branch.

Neither is open source, and the Fusion comparison's line that both of those products are needs correcting on blizzard's
side. Soleur moved to the Business Source License early in the year to protect a hosted launch, and later had to scrub
"open source" from its own site. Blizzard's repositories are public with no license at all, which leaves them all rights
reserved. A prospect weighing the two should hear that from us before they find it.

The honest summary is that Soleur has built the most elaborate version of the layer blizzard leaves below it, and has
learned, one incident at a time, why someone would want a layer above.

## What the differences ask

Four decisions fall out of this comparison, and one of them is urgent.

**Confinement is the urgent one.** The shipped default runs every worker with permissions bypassed, which is the
condition [`milestone:polyglot`](../../../milestones.md) promises to end, and the Landlock boundary the Fusion
comparison relied on to soften that gap does not govern live workers. Soleur demonstrates that a fail-closed sandbox
with a network allowlist is buildable by one person and can be proven on every deploy. The worker-security slice of
[`epic:security`](../../../epics.md) currently frames the work as harness configuration plus one autonomy level. Either
that framing meets the milestone through a safer default and a narrower permission posture, or the slice needs an
operating-system boundary added to its scope. Which of the two is the decision. The Fusion comparison's sandboxing row
should be corrected in the same change.

**Learning deserves a decision, not a copy.** Agent memory is now the idea from two competitors that the mission does
not refuse and the registry does not hold. Soleur's evidence argues against the automatic half: write-back swelled its
prompts, and its retrieval lost to grep. The half worth taking is the one blizzard's workspace already does by hand,
which is to cluster what went wrong across many chunks, draft the fix to a rule or a prompt, and let a human decide. The
likely answer is to make that pass recurring once [`epic:cadence`](../../../epics.md) exists, not to give agents a
memory. It should be a considered answer either way.

**The methodology seam needs a written boundary.** Soleur is proof that a lane can be portable and sold on its own,
which is the best news the delegation thesis could get. It is also a lane that wants to do the fleet's work. Its
lifecycle opens the pull request, merges it, polls the release, and schedules its own follow-ups, so a repository
carrying Soleur would collide with blizzard's deliver node unless those stages were turned off. Blizzard already denies
its workers the harness's scheduling tools. Saying, in one place, what a methodology running inside a station may not do
(merge, open pull requests, schedule, choose its own next work) is what would let an off-the-shelf methodology be
dropped into a repository blizzard drives.

**The landing's red rate is worth a second look.** After adopting a merge queue that lands one green entry at a time,
Soleur's default branch failed 8 of its last 100 CI runs, a short window, and its bot fixes revert themselves when they
turn the branch red. The [field survey](../factory-field/index.md) puts blizzard at about one landing in five turning
`master` red, through a strict single-file queue. The cause is not settled by this comparison, but the asymmetry is
large enough to examine before [`epic:advanced-delivery`](../../../epics.md) designs its train, because the train is
where the lesson would land. Whether delivery owns a revert path belongs in the same conversation.

The remaining differences are the design working. Departments beyond engineering, a priced multi-tenant service, and a
model that picks which issues travel together are positions the mission already takes, and Soleur's own competitive
analysis reaches the matching conclusion from the other side: an engineering fleet orchestrator is a different customer,
a source of ideas rather than a rival.
