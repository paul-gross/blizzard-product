---
epic: runner-host
refinement: scaffolded
slices:
  - name: full
    status: horizon
---

# Plan — `epic:runner-host`

An operator who pays for two plans — one with Anthropic, one with OpenAI — wants one machine to spend both. Today the
only way to get that is to run the machine twice: two runner daemons side by side, each with its own directory, its own
port, its own service unit, and its own carefully disjoint slice of the environments, because a runner is the smallest
thing that can be paused. When the Anthropic window runs dry at two in the morning, the operator wants the Claude work
to stop and the OpenAI work to carry on; the only brake that scopes that narrowly is a whole second installation. The
platform has quietly made *the process* the unit of capacity, and every new plan an operator buys costs them another
copy of it.

The same operator building three products meets the same wall from the other side. A project is a first-class idea at
the hub, and a runner can name the projects it serves, but a runner still has exactly one workspace. Three projects
whose repositories do not live together mean three runners in three directories, and the desk full of stacks that
`milestone:projects` set out to collapse collapses only as far as the hub.

This epic separates the things a runner has been doing at once. What claims work, holds it, and spends a subscription on
it stays a **runner**, exactly the concept the hub already knows: an identity with its harnesses, its model mapping, its
worker security, and its own brake. What owns the machine becomes the **runner host**: the one daemon per machine that
holds its workspaces and their environments, holds the subscriptions the operator pays for, and runs as many runners as
they care to declare. A machine with a Claude runner and an OpenCode runner is then one host, one environment pool, two
subscriptions, and two runners that each stop on their own; a machine working three projects is one host with as many
workspaces as those projects' repositories need.

The hierarchy stays deliberately shallow. A host belongs to one tenant, always, so every runner on it works in the same
world. The hub learns the host as a thing of its own, but it still grants a chunk to exactly one runner, and that runner
still holds the chunk's route, tenure, and leases from claim to landing, so exactly-once, pause, eligibility, and
recovery keep the shape they have. What is new is a host above the runners, known to the hub and to the board,
workspaces shared among its runners, and subscriptions as a concept of their own that runners draw on rather than
declare.

The machine and the hub change together, as one piece of work. A host the hub could not see would need a runner to sign
the operator in on its behalf, a registration that pretends a runner still has one workspace, and every sample of a plan
copied onto each runner that spends it — each a stand-in built only to be replaced.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## Three things a runner names

A runner may name one **subscription**: the plan its work is paid from. A subscription is declared once on the host, and
any number of the host's runners may spend it — one OpenAI plan feeding an OpenCode runner and a Codex runner — but a
runner never spends two. The host samples each subscription once, however many runners draw on it. When a harness
reports that a plan's window is spent, the brake engages on every runner spending that plan and on no other, and it
stops the whole runner, exactly as the runner's own brake does today; an operator still lifts it by hand.

Naming a subscription is the operator's word about which plan a runner's work is paid from, not a wire blizzard runs.
The harnesses a runner launches still sign in however the operator configured them, and blizzard never hands a worker a
plan's credentials. So a runner enabling Claude Code and OpenCode under one subscription is as valid as any other: the
operator has said both draw on that plan, and a limit reported by either brakes the runner and every runner spending it.
A subscription is an optional addition that lets the fleet track a plan's usage and pace against it; a runner working on
API keys names none, and works exactly as well.

A runner names the **projects** it serves, and binds each of them to one **workspace**. A workspace is a place work
happens — a root, a named set of environments within it, and its standing prompt — and it may serve several projects and
several runners at once, as one workspace holds both `blizzard` and `blizzard-context`. A project, though, has exactly
one workspace for any runner that serves it. That is what spares a runner a choice it has no grounds to make: a chunk
belongs to one project, so the runner that claims it already knows the one place its work will happen.

The binding is also where the operator says how much of that work may run at once. Each runner, project, and workspace
together carry their own count of agents working in parallel, and it is a separate thing from how many environments the
workspace has. Environments are the room; agents are the people in it. A workspace with twelve environments may be
worked by four agents at a time, leaving the rest for another runner; a binding allowed twelve agents in a workspace of
four environments is a valid one, if not a useful one, since an agent with no free environment simply waits for one.

How the room is divided is the operator's choice. One directory on disk may be declared as two workspaces, each naming
its own environments — `r1` to `r6` for the Claude runner, `oce1` to `oce4` for the OpenCode runner — so that each
runner keeps a set of its own inside one checkout. Declared instead as one workspace of ten environments, both runners
would draw on the same pool, and that works equally well.

What a workspace holds is not blizzard's concern. A workspace is provided by whatever stands behind blizzard's workspace
seam, and whether its environments already hold every repository a project links, or clone one when a chunk first needs
it, is that provider's business. Blizzard checks nothing about it when a project is bound.

## A host the hub knows

A host is added at the hub, as a runner is today: the operator runs `hub host add` once, in the tenant the host will
serve, and receives the host's id and a credential of its own, shown once. From then on the machine looks after itself.
Bringing a runner up is something the host asks the hub for under its own credential: the hub mints the runner's id and
token in the host's tenant and under the host, and the host writes both into its configuration. Neither the operator nor
an agent working on the machine needs a sign-in at the hub to add a runner — only the host's credential, which already
lives on the machine the runners will run on. A host the hub has not been told of yet still runs, holding the one runner
it graduated from, which talks to the hub as it always has.

The credential is deliberately narrow. It lets the host describe itself — its workspaces, its subscriptions, the address
its panel is reached at — report its subscriptions' usage, sign the operator in to its panel, and add or retire runners
under itself. It cannot claim work, and it cannot act as any of its runners: each runner keeps its own token, and the
hub still knows a runner's work by that token alone. Retiring a host revokes its credential and retires every runner
under it.

## What to build

- **One host, many runners.** A single daemon runs every runner the operator declares on the machine, each holding the
  id and token the hub issued for it. A machine that runs two installations today runs one, on one port, under one
  service unit. A runner that cannot work — a token the hub refuses, an identity that is not its own — stops alone, and
  its neighbours carry on.
- **One store for the machine.** A host keeps one database holding every runner it runs, each runner's records marked as
  its own and read only through that runner's view of the store. What spans runners — the environments a workspace has
  handed out, a plan's exhaustion braking every runner that spends it — is settled in one transaction rather than
  stitched across files, so two runners can never hold one environment and a brake never reaches half its runners.
- **The host as a hub record.** A host is a record in its tenant with an id the hub mints, a name, and a credential.
  `hub host add` creates it, `hub host retire` retires it with its runners, and an existing runner can be adopted into a
  host without losing its id or token.
- **Runners added at the host.** `blizzard runner add <name>` on the machine asks the hub, under the host's credential,
  for a new runner under the host, and records its id and token in the host's configuration. `hub runner add --host`
  does the same from an operator's session.
- **A host serves one tenant, for life.** A host is added in one tenant and every runner it adds is born there; the hub
  refuses to place a runner of another tenant under it. Nothing moves a host between tenants: its operator retires it
  and adds another, as a runner's store is never migrated either.
- **Each setting where it belongs.** The host holds what is about the machine: the hub it talks to and its credential
  there, the address the operator's browser reaches it on, sign-in, tracing, its workspaces, and its subscriptions. A
  workspace holds its root, the environments it names within that root, and its standing prompt. A runner holds what is
  about the work it takes and how its workers run: its hub token, its harnesses and models, its worker security, its
  gates and spend ceiling, the subscription it spends, and the projects it serves. Each binding of a runner's project to
  a workspace holds how many agents may work it at once.
- **Subscriptions as the host's own concept.** The operator declares each plan once on the host, and a runner may name
  the one subscription it spends. The host samples each plan once, however many runners draw on it, and reports it to
  the hub once, as the host's.
- **An exhausted subscription stops only what spends it.** When a harness reports a usage limit, the runner's own brake
  engages on every runner of the host spending that subscription and on no other, so the Claude work parks while the
  OpenAI work carries on. The brake stops the whole runner, as it does today, and only an operator lifts it, one runner
  at a time — from the panel, the CLI, or an agent driving the CLI. The reset time it names is read from that
  subscription's samples alone.
- **Projects bound to workspaces, with their agents.** A runner binds each project it serves to exactly one of the
  host's workspaces, with the count of agents that may work it at once. The runner declares its bindings to the hub, and
  when it claims a chunk it already knows the one workspace the work will happen in. A chunk of a project the runner
  does not bind is passed over.
- **One environment pool per workspace, shared.** A workspace's environments belong to the workspace, not to any runner,
  and the host arbitrates them among every binding to it.
- **One panel, one sign-in.** The host serves one origin, and the operator signs in to it once, with the host as the
  hub's sign-in client. Its panel shows the machine — workspaces, environments, subscriptions — and each runner's view
  within it, with each runner's status, brake, and operator verbs scoped to that runner as they are today. A runner's
  routes live under the host's address for that runner; nothing answers at the paths a lone runner used.
- **The machine seen as one thing.** The board shows a host with its runners inside it rather than unrelated peers that
  happen to share a desk, and a plan's usage once, on its host, with each runner that draws on it named.
- **Every runner graduates to a host.** Deploying the release turns each existing runner into a host of one: its
  database migrates forward into a host's, holding that one runner, and its configuration file is rewritten as the
  host's — a short, mechanical step that is part of the deploy, which an agent carries out as readily as a person, so
  blizzard ships the procedure rather than a command for it. The runner keeps its id, its token, and every chunk it
  holds, and until its host is registered at the hub it works there exactly as a runner with no host does today.
- **Ingest gathers runners into one host.** A registered host takes in a runner from another host's database — or from a
  runner not yet graduated — copying its records across, re-numbering what must be re-numbered, and removing it from the
  source, which is set aside once empty. Two runners on one desk become one host by graduating both and ingesting one
  into the other, and the same verb serves any later rearrangement.

## Where the runner goes next

The runner is the level every per-machine lever already speaks to, so the levers attach there unchanged:
`epic:throttling` paces a runner against the subscription it spends, and `epic:tagging` lets a runner declare the work
it takes. A runner that can no longer serve a chunk because its subscription is exhausted is `epic:resilience`'s problem
to ride out, not this epic's.

## Left for later

The brake does not lift itself at the reset time a harness reported, and an exhausted plan is not cleared once for every
runner that spends it. Both would change what the brake promises today — that only an operator lifts it — and this epic
keeps that promise while it changes what the brake covers. Nor may a runner bind one project to two workspaces; a rule
for choosing between them on every claim earns its place only once someone needs more than one.

Nor does a subscription reach the workers that spend it. Handing a runner's harnesses the credentials of the plan it
names, so that the declaration becomes the wire, is a step of its own; until then the operator's tool configuration
decides which plan a worker actually draws on.
