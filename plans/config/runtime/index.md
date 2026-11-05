---
epic: config/runtime
refinement: scaffolded
status: horizon
---

# Plan — `epic:config/runtime`

[`persona:harness-engineer`](../../../charter/personas/harness-engineer.md) tunes a fleet the way a mechanic tunes an
engine: move one thing, listen, move it back. Today every one of those moves on a runner costs a restart of the host it
lives on, and that restart is not free. It interrupts every session each of the host's runners is carrying, each of
which has to find its way back to where it was cut off, so the engineer who wants to lower a budget at ten at night, or
hold a runner to fewer agents while they watch a flaky chunk, either waits for the whole host to go idle or pays for the
experiment in broken sessions. Adding a runner to a host is the same: the new runner waits for the host's next start.
The hub is kinder only by degree: its work sources, repositories, and secrets already change while it runs, since
`epic:live-config`, but the settings that govern how it behaves — how often its background passes run, how much
transcript it will accept, how hard it throttles a login — are read once when it starts and held until it stops.

Nothing about a runner needs it to be this way. The host reads its file once when it starts and freezes what it read
into the objects it builds, and no path was ever built to change them afterwards. This epic builds that path, for the
runner first and the hub beside it: it makes the operational settings of both daemons something an operator changes
while they run, and makes the moment a change takes hold something the operator can predict rather than discover. It is
the companion of `epic:config/sweep`. The sweep decides *what* is configurable — it finds every constant somebody chose
once and promotes it, with an account of what moving it does. This epic decides *when* a change reaches the running
fleet. Neither depends on the other to land, but each is worth more beside the other: a constant the sweep promotes is
only half a knob if turning it means a restart, and a setting this epic makes live is only as useful as the list of
settings that exist.

The two daemons are different machines with different owners, different failure modes, and different things at stake in
a restart, so the epic is cut into a runner half and a hub half, and each daemon owns its own settings outright. The
hub's settings are changed through the hub's API; a runner's are changed through that runner's API, and stay on the
runner. Neither daemon holds the other's, and neither file is a way to change a running daemon. The runner half is what
the epic is for, because that is where a restart hurts, and it lands first. The hub half is worth having and follows it.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic, or looking up a setting's lifetime |

## Lifetimes

Every setting has a **lifetime**: how long a value, once loaded, is lived with before the setting is evaluated again.
The lifetime is what an operator reads to know when a change they make will bite, and it is the vocabulary this epic,
its spec, and every conversation about a setting use. There are four.

Both daemons work the same way at heart: they loop, and each time round they make a pass over what is in front of them.
That rhythm gives most settings their answer, and the rest are the exceptions to it:

- **`pass`** — evaluated at the start of every pass, and lived with until the pass ends. A pass loads its settings once
  and works the whole way through from that one picture; a change made while it runs waits for the next pass. Load the
  settings, make the pass, load the settings again. Nothing is read halfway through, so no pass ever weighs a budget it
  read at the start against a pace it read at the end. A request to the hub is a pass of its own, evaluating its
  settings when it arrives.
- **`session`** — evaluated when a worker is launched, and lived with for that worker's whole session. A worker is the
  coding agent a runner launches into a session to work one step, and it is handed its model, its prompt, its
  environment, and its harness settings when it starts; nothing reaches inside a running process to change them. A
  change reaches every session launched after it, and a session already running keeps what it launched with, because
  changing a worker's world under its feet is a different and worse kind of surprise than changing it between sessions.
- **`boot`** — evaluated when the daemon starts, and lived with until it stops. These are a daemon's footing rather than
  its behavior — for a runner host, the address and port it listens on, the store it writes to, the hub it reports to
  and the token it presents there — and changing one is a restart of the daemon, honestly said. A `boot` setting is the
  only kind an environment variable may supply, and where one does it overrides the file: an environment variable is
  read once, when the process starts, which is exactly a `boot` lifetime. A `pass` or `session` setting is never read
  from the environment, because a value fixed at process start would quietly make a live setting a `boot` one.
- **`immutable`** — evaluated once, when the thing it belongs to is created, and never again, not even across a restart.
  A runner's id is the one: changing it would make a different runner.

A few values are not settings at all and have no lifetime: limits a protocol, a peer, or another setting depends on,
which are pinned in code rather than offered.

Each setting declares its lifetime in the same registry that declares its type and its default, so the contract
documents that already pin every key carry it too, and the answer to "can I change this live, and when will it bite?" is
a lookup.

## The runner

A host serves its runners from one process, on one address, and reads its file once, when it starts. What it reads it
freezes: into the context each runner's loop caches on its first tick, into the timer each loop runs on, into each
runner's harness adapters and usage samplers, and into the set of runners it starts. Nothing it does afterwards looks at
the file again, the hub sends it back nothing but the fleet-wide pause, and the loop's own tick is not even in the file:
it is an environment variable read at boot.

- **The runner's settings in its own store.** Every runner setting whose lifetime is `pass` or `session` becomes an
  entry in the host's store, revised rather than overwritten, with a log of each change and who made it. The store, not
  the file, is the answer to what a runner is configured with.
- **Changed through the runner's API.** The runner's local API reads and writes those entries, and its CLI and its local
  page speak that API. A write is checked against the same rules a boot applies to the file, in full, before any of it
  applies: a write that would be refused at boot is refused, and the runner keeps running exactly as it was, saying what
  it rejected and why. A runner needs nothing from the hub to change its own settings, so it can be tuned while the hub
  is out of reach.
- **The file seeds what the store lacks.** At startup, every setting the file states that has no entry in the store yet
  is created from the file. A setting the store already holds is left alone, and the file's value for it is ignored. The
  file is how a new runner, or a setting new to an existing one, gets its first value; after that the API owns it. An
  operator who keeps their host file in version control still stands a fresh runner up from it.
- **Throttling, spending, and capacity live for one tick.** Their lifetime is `pass`, and the tick is the pass that
  reads them. The throttle's budget, pace, schedule, and utilization floor, what the runner does on a decline or a usage
  limit, the cost caps per chunk and per runner, how many agents it runs, which gates it holds, which projects it
  serves, and whether its queue is strict are all loaded at the start of the next tick and held for the whole of it.
  These are the settings the engineer reaches for in the middle of a night, and a tick is short enough that the next one
  is soon. The runner tells the hub what it now accepts at that tick's registration, so the queue the hub hands out
  reflects it. Lowering a runner's agents never kills a session to make the number true: the runner stops claiming until
  it is under the line.
- **Every pass, not only the tick.** Each runner runs its tick and, where tracing is on, a trace sweep beside it, and
  the host runs one tick of its own across all its runners — the pass that reports to the hub, samples subscription
  usage, and starts and stops runners. Each evaluates its own `pass` settings at its start, and so does each step inside
  it, the transcript drain and retention among them.
- **What a worker is handed lives for its session.** Its lifetime is `session`. Model and effort aliases, the harness
  settings, a worker's environment, its shell and path, and the runner and workspace prompts apply to every session
  started after the change, and never to one already running. A session records which settings it launched under, so the
  engineer comparing two runs of the same chunk can see which side of an edit each fell on.
- **The loop's own cadence as a setting.** The tick's interval becomes a setting like any other, changed through the
  runner's API and seeded from the file. Its lifetime is `pass`: each tick evaluates the interval to the next, so a
  runner can be sped up or slowed down while it runs. The environment variable that sets it today is deprecated, since
  an environment variable may supply only a `boot` setting; until it is removed, it seeds the entry only when neither
  the store nor the file holds one.
- **Every setting of a runner, its name included.** Nothing inside a runner's own configuration waits for a restart. The
  name is a label the runner declares at each registration, not its identity, so its lifetime is `pass` like the rest.
- **A runner's id never changes.** Its lifetime is `immutable`. The hub mints it when the runner is added, every lease
  and record the runner holds is keyed by it, and every worker reaches its runner by a path that carries it. A runner
  wanted under another id is retired and a new one added.
- **Runners that come and go on a running host.** A runner added to a host starts at the host's next tick, and one
  retired from it stops there, with no restart of the host or of the runners beside it. Retirement already stops a
  runner without a restart, because the host notices the hub refusing the retired runner's token; adding one waits for
  the host's next start today.
- **The host's footing lives for the host's boot.** Its lifetime is `boot`. The host's address and port, which every
  running worker was handed to reach its runner by; its store, where every other setting now lives; the hub it reports
  to and the host token it presents there; and its auth are the ground every runner on the host stands on. They are not
  entries: they stay in the file, read at boot as today, and changing one restarts the host. They rarely move, and
  moving them under running sessions is the one change that cannot be made safe cheaply.
- **Workspaces and subscriptions as records.** The workspaces a host provisions into and the subscriptions its runners
  spend become records of the host, the way forges and repositories are records of the hub: created, edited, and retired
  through the host's API, never deleted, and taken up at the host's next tick. A workspace's id and a subscription's
  slug and provider are `immutable` — the hub records the workspace a claim worked in, and usage history and limit marks
  are kept by subscription — so a workspace or subscription wanted under another id is retired and a new one created.
  Retiring one stops new work through it: the host stops reporting it, and the hub already withholds work from any
  runner binding that names a workspace or subscription the host does not report. Its history stays.
- **A workspace is the operator's to reshape.** Every setting of a workspace other than its id — its provider, its root,
  its repositories, its environment pool and limit, its base branch — can change at any time, with nothing refused, and
  applies to the next environment acquired. Blizzard does not reconcile what is on disk: an operator who repoints a
  workspace adjusts the disk to match, outside blizzard, and an environment already held is released and resumed through
  the workspace as it now stands.
- **The settings in force, visible.** The runner's status, its local page, and the hub's view of the fleet show which
  revision of its settings each runner is running and when it took hold. "What is this runner actually running with" is
  answered by asking the runner, not by reading a file the runner may have outgrown.

## The hub

The hub already knows how to change configuration while it runs. `epic:live-config` gave it records that are revised
rather than overwritten, a log of every change and who made it, an API its CLI and board both speak, and a board to read
and edit them on. What it does not yet cover is the hub's own behavior, which still lives in `blizzard-hub.toml`.

- **The hub's operational settings as records.** The cadence and budgets of the hub's background passes, its transcript
  caps, the egress and trace-export settings, the login throttle, and the produces and route-token modes become
  configuration held in the hub's store and changed through the hub's API, under the same revisions, change log, and
  board that work sources already have.
- **Passes that pick up their settings each time round.** A background pass loads its interval, its budget, and every
  other setting it uses when it starts, not when the hub started, and holds them until it ends. A request does the same
  when it arrives.
- **The file seeds what the store lacks.** The rule is the runner's: at startup, a setting the file states with no entry
  in the store is created from the file, and one the store already holds is left alone. The hub's file keeps, as its
  own, what is genuinely the process's: the address it listens on, its store, its auth providers, and the key its
  secrets are sealed under.

## Across both

A few settings only make sense in pairs that span both daemons. The hub decides a runner is offline after a silence that
must outlast the runner's own heartbeat; it decides a usage sample is stale after a window that must outlast how often
the runner takes one; a transcript record the runner is willing to send must fit under what the hub is willing to take.
At boot each half trusts the other's defaults. Once either can change live, nobody is guarding the pair, and since each
daemon owns its own side, neither can refuse the other's change. The hub, which hears from every runner, checks each
pair whenever a runner reports its settings or the hub's own move, and flags the runner whose settings break it rather
than letting the fleet misbehave quietly.

## What this epic does not do

It does not decide which constants become settings; that is `epic:config/sweep`, and every setting the sweep promotes
arrives declaring its lifetime. It does not let either daemon hold or change the other's settings: the hub does not tune
a runner, and a runner does not tune the hub. It does not reach into the harnesses' own configuration, which is theirs.

## Open questions

- Whether a session's settings revision is recorded as a fact on the hub, which is what makes comparison across nights
  trustworthy, and which `epic:config/sweep` asks of its comparable run too. The two epics should answer it once,
  together.
- Whether the runner's settings API is open to the same users its local page already admits, or to a narrower role,
  since a setting such as a worker's environment reaches every session it launches.
- Whether a declarative apply, of the kind `epic:live-config` gave the hub's records, is wanted for either daemon's
  settings beyond the startup seed.
