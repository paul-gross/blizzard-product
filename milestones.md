# Milestones

What users will be able to do. A milestone is a destination stated in the user's terms, and it comes first: you declare
where the product must reach, then ask what work the journey requires — the milestone demands its epics, never the other
way around.

| Milestone                      | What users will be able to do                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `milestone:polyglot`           | Run the fleet on the coding harness of their choice — Claude Code, Codex, or OpenCode, first-class and mixable by node — with the safeties on: no worker runs with permissions dangerously bypassed.                                                                                                                                                                                                                    |
| `milestone:projects`           | Run every project from one fleet: a single hub hosting many projects and the sources they draw from, and a single runner host per machine working all of them — a workspace for each project or set of projects, not a stack per project. Each project lands and deploys its finished work its own declared way.                                                                                                        |
| `milestone:hardening`          | Decide for themselves how the fleet behaves: every operational constant theirs to set, what each runner will take and when and at what rate theirs to declare, a provider outage ridden out rather than slept through, nothing growing without end underneath them, every upgrade taken when they choose and told beforehand what it asks of them, and every question about the platform answerable without cloning it. |
| `milestone:human-in-the-loop`  | Stop being the wire between the fleet and everything it needs: it reaches them wherever they are when a decision is genuinely theirs, and settles CI's verdict itself when it is not.                                                                                                                                                                                                                                   |
| `milestone:mobile`             | Carry the fleet in a pocket: watch the night, answer a question, and unblock a chunk from a phone, through notifications that arrive the way the phone's own do.                                                                                                                                                                                                                                                        |
| `milestone:project-management` | Assemble what the fleet works on from inside blizzard — browse the backlog, take many items at once, and shape them into chunks — instead of handing over ids one at a time.                                                                                                                                                                                                                                            |
| `milestone:future-state`       | Unsorted work that belongs to no major milestone underway.                                                                                                                                                                                                                                                                                                                                                              |

## `milestone:polyglot` — any harness, with the safeties on

No single coding agent should ever be load-bearing for the fleet. The operator runs the harness that serves the work —
chosen for the whole fleet, or mixed by node the way models already are — and when a harness ships a new major version,
the fleet takes it up beside the old one rather than forcing a migration. Harness choice becomes a tunable rather than a
fact of the platform: the harness engineer runs the same chunk through two harnesses and compares them on cost and
quality.

Choice never costs safety. The operator sets each runner's terms once — how much autonomy its workers have, and which
native configurations and tool rules they carry — and every harness honours those terms in its own way, without leaving
a headless run stuck on a prompt and without any worker running with its permissions dangerously bypassed. Protected
branches remain the forge's to guard, not the agent's to respect on a bad night.

| Epic                            | Status    |
| ------------------------------- | --------- |
| `epic:adapters/opencode`        | delivered |
| `epic:adapters/opencode-2`      | horizon   |
| `epic:adapters/codex`           | horizon   |
| `epic:security/worker-security` | delivered |

## `milestone:projects` — one fleet, every project

One operator should be able to run every project they care about from one fleet. Today blizzard silently assumes it
serves only itself, so a second project means a second hub, a second runner, a whole second stack on the same desk, none
of them aware of the others. This milestone makes the project a first-class idea — what to do and who does it, grouped —
so one hub serves many projects, one runner host per machine works all of them, and one board shows the whole fleet or
narrows to the single project that matters right now.

Each project keeps the terms of its own work: the sources it draws from, the repositories it may land in, and how
finished work is delivered and deployed there — merged by the fleet or by a human, deployed in the order its
repositories depend on one another, proved healthy or rolled back. Above projects sits the hard boundary, the tenant:
one hub can host wholly separate worlds, each with its own graphs, projects, runners, and board, and none aware of the
others.

| Epic                              | Status    |
| --------------------------------- | --------- |
| `epic:live-config`                | delivered |
| `epic:multi-tenancy/hub`          | delivered |
| `epic:projects/hub`               | delivered |
| `epic:projects/runner`            | delivered |
| `epic:multi-tenancy/runner`       | delivered |
| `epic:runner-host`                | delivered |
| `epic:projects/host`              | retired   |
| `epic:advanced-delivery/actions`  | delivered |
| `epic:advanced-delivery/forges`   | delivered |
| `epic:advanced-delivery/flows`    | delivered |
| `epic:advanced-delivery/policies` | delivered |
| `epic:advanced-deployment`        | horizon   |

## `milestone:hardening` — the platform stops guessing on your behalf

A young platform is full of choices someone made once, on the operator's behalf, without knowing their fleet, their
machine, or their budget. Hardening hands those choices back. What was a constant in code becomes a decision the
operator makes, and what the platform presumed — that the provider is always reachable, that storage is free, that an
upgrade may land the moment it is ready, that a reader will clone the repository to learn it — becomes something it is
built to handle.

The same intent turns inward, onto the ground the fleet itself stands on: code an agent meeting it for the first time
can find its way through, tests that gate every landing quickly and prove what they claim, and releases that reach the
people running blizzard when they choose. Each holds because the build enforces it, not because someone remembers to.
Little of this is new capability; it is what a platform does after it works, and the release that reaches this milestone
is blizzard's 1.0.

| Epic                                     | Status      |
| ---------------------------------------- | ----------- |
| `epic:runner-host`                       | delivered   |
| `epic:release-engineering/first-release` | delivered   |
| `epic:release-engineering/compat-floor`  | delivered   |
| `epic:release-engineering/notes`         | delivered   |
| `epic:release-engineering/release-lines` | delivered   |
| `epic:release-engineering/dark-launch`   | delivered   |
| `epic:release-engineering/supply-chain`  | delivered   |
| `epic:storybook`                         | delivered   |
| `epic:throttling`                        | delivered   |
| `epic:tagging`                           | horizon     |
| `epic:resilience`                        | horizon     |
| `epic:retention`                         | horizon     |
| `epic:config/model-resolution`           | horizon     |
| `epic:config/sweep`                      | horizon     |
| `epic:ui-toolkit`                        | horizon     |
| `epic:documentation`                     | horizon     |
| `epic:test-suite/foundation`             | delivered   |
| `epic:test-suite/hub-tier`               | delivered   |
| `epic:test-suite/runner-tier`            | delivered   |
| `epic:test-suite/fleet-tier`             | delivered   |
| `epic:test-suite/journey`                | delivered   |
| `epic:test-suite/controlled-clock`       | in-progress |
| `epic:test-suite/optimization`           | horizon     |
| `epic:architectural-sweep`               | delivered   |
| `epic:strict-types`                      | in-progress |
| `epic:adopt-di`                          | horizon     |

## `milestone:human-in-the-loop` — the operator stops being the wire

Work stops and waits for a person for two quite different reasons, and today they look identical from the outside.

In the first, the fleet has met something only a human can settle — an ambiguity in what was asked, a judgment call the
charge does not cover — and it is right to wait. What is wrong is *how* it waits: silently, on a board nobody is looking
at, until the operator next walks past a terminal. A chunk that stopped at nine in the evening has spent the night
achieving nothing, and the fleet's whole promise is the night. So a question goes and finds its person, arriving as a
message on a chat app they already have open and answering in a tap.

In the second, nothing about the moment needs a human at all. CI has spoken, or a reviewer has left comments, and that
verdict is sitting somewhere the session that produced the work cannot see. A person reads it, understands it, and types
it back in — acting as a wire between two machines, adding nothing but latency. Routing those results back into the
owning session closes the loop without them, and the cap on how many rounds it may spend is the part that matters: an
agent arguing with CI indefinitely is worse than one that stops and asks, which returns the moment to the first kind
honestly.

The milestone is named for the distinction. A human belongs in the loop exactly where judgment lives, and nowhere else.

| Epic               | Status  |
| ------------------ | ------- |
| `epic:chat`        | horizon |
| `epic:ci-feedback` | horizon |

## `milestone:mobile` — the fleet in a pocket

The fleet works while its operator is away from the desk. That is the proposition, and it is also precisely when they
are least equipped to answer it: the surfaces that could unblock a chunk are a browser tab on a closed laptop and a
terminal in another room.

A phone changes that arithmetic completely. Not because anyone wants to run a fleet from a phone — nobody does — but
because the operator's part in a good night is small and interruptive. Read what the fleet is asking, decide, and put it
away. That is a phone-shaped job, and it is currently a laptop-shaped one.

How the fleet reaches the phone — an installed web app for the one hub most operators run, or native apps that follow
every hub and runner the operator watches as one view, with notifications as reliable as the phone's own — is
deliberately left open, and whichever ships first settles it for the rest.

Either way, it stands on the notification fan-out that `milestone:human-in-the-loop` builds. Reaching a person is one
problem, and it is solved once.

| Epic           | Status  |
| -------------- | ------- |
| `epic:pwa`     | horizon |
| `epic:android` | horizon |
| `epic:ios`     | horizon |

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
of what an item says; this is a surface for moving items into the fleet, not for authoring them.

It arrives after `milestone:projects` for a plain reason — a hub serving many projects that draw on many sources changes
the shape of the screen enough that building it first would mean building it twice.

| Epic          | Status  |
| ------------- | ------- |
| `epic:intake` | horizon |

## `milestone:future-state` — unsorted work

Work that belongs to no major milestone underway, held here until one claims it.

| Epic                                      | Status  |
| ----------------------------------------- | ------- |
| `epic:cadence`                            | horizon |
| `epic:steering`                           | horizon |
| `epic:signals`                            | horizon |
| `epic:preview`                            | horizon |
| `epic:worker-profiles`                    | horizon |
| `epic:provider-kit`                       | horizon |
| `epic:terraform-provider`                 | horizon |
| `epic:tenant-telemetry`                   | horizon |
| `epic:demo`                               | horizon |
| `epic:queue/priority`                     | retired |
| `epic:urgency`                            | horizon |
| `epic:chokepoint/chokepoint`              | horizon |
| `epic:chokepoint/epic-dwf`                | horizon |
| `epic:code-harness-window/agent-observer` | horizon |
| `epic:code-harness-window/swarm-view`     | horizon |
| `epic:mutation-testing/angular`           | horizon |
| `epic:lab/bench`                          | horizon |
| `epic:lab/experiment`                     | horizon |
| `epic:lab/lab-runner`                     | horizon |
| `epic:lab/specimen`                       | horizon |
| `epic:lab/verdict`                        | horizon |
| `epic:lab/lab-board`                      | horizon |
| `epic:lab/seeded-defects`                 | horizon |
| `epic:lab/evidence`                       | horizon |
| `epic:advanced-delivery/gitlab`           | horizon |
| `epic:advanced-delivery/trains`           | horizon |
| `epic:ui-redesign`                        | horizon |
