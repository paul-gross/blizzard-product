---
epic: runner-host
refinement: scaffolded
slices:
  - name: runner
    status: horizon
  - name: hub
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

This epic separates the two things a runner has been doing at once. What claims work, holds it, and spends a
subscription on it stays a **runner**, exactly the concept the hub already knows: an identity with its harnesses, its
model mapping, its agent limit, and its own brake. What owns the machine becomes the **runner host**: the one daemon per
machine that holds its workspaces and their environments, holds the subscriptions the operator pays for, and runs as
many runners as they care to declare. A machine with a Claude runner and an OpenCode runner is then one host, one
environment pool, two subscriptions, and two runners that each stop on their own.

The hierarchy stays deliberately shallow. The hub's vocabulary does not grow in the first slice: it still grants a chunk
to exactly one runner, and that runner still holds the chunk's route, tenure, and leases from claim to landing, so
exactly-once, pause, eligibility, and recovery keep the shape they have. What is new lives on the machine — a host above
the runners, and subscriptions as a concept of their own that runners draw on rather than report.

## What to build — the runner slice

- **One host, many runners.** A single daemon runs every runner the operator declares on the machine, each registering
  with the hub under its own id, so a machine that runs two installations today runs one.
- **Subscriptions as the host's own concept.** The operator declares each plan once on the host, and each runner names
  the subscription it spends. Several runners may spend the same subscription — one OpenAI plan feeding an OpenCode
  runner and a Codex runner — and the host samples each subscription once, however many runners draw on it.
- **An exhausted subscription stops only what spends it.** When a harness reports a usage limit, the brake engages on
  every runner drawing on that subscription and on no other, so the Claude work parks while the OpenAI work carries on.
- **One environment pool, shared.** The host owns its workspaces and their environments and arbitrates them among its
  runners, so the operator stops dividing a pool by hand and a runner that is idle leaves its share free for one that is
  not.
- **Workspaces, plural, from the start.** The host's configuration holds a list of workspaces, each with its own root,
  environments, and prompt, even while a machine has only one. `epic:projects` gives each workspace a project; shaping
  the list now means that epic adds an entry rather than reopening how a host is configured.
- **The existing machine, carried over.** A runner configured the way it is today becomes a host with one runner under
  the same id, without re-enrollment or a stranded chunk, and two runners already on one machine fold into one host
  keeping both ids.

## What to build — the hub slice

- **The host as something the hub knows.** Runners register under the host they belong to, so the board shows a machine
  as one thing with its runners inside it rather than as unrelated peers that happen to share a desk.
- **Subscriptions shown once, where they live.** A plan's usage appears on its host, with each runner that draws on it
  named, rather than repeated on every runner that spends it.
- **Enrollment at the host.** Bringing a new runner onto an enrolled host is a configuration change on the machine,
  never a window in which the hub relaxes its enforcement for the whole fleet.

## Where the runner goes next

The runner is the level every per-machine lever already speaks to, so the levers attach there unchanged:
`epic:throttling` paces a runner against the subscription it spends, `epic:tagging` lets a runner declare the work it
takes, and `epic:projects` records the projects a runner serves. The host is what `epic:projects` means by one runner
host per machine: its runner slice binds each of the host's workspaces to a project, and builds on this epic rather than
beside it. A runner that can no longer serve a chunk because its subscription is exhausted is `epic:resilience`'s
problem to ride out, not this epic's.

## Open questions

- Whether a runner spends exactly one subscription, or one per harness it binds. A graph that mixes harnesses by node
  needs a runner able to serve all of them, and a chunk stays on its runner from claim to landing.
- Which settings belong to the host, which to each workspace, and which to each runner: worker settings, human gates,
  and the web surface's origin are candidates for the host; the workspace prompt belongs to its workspace; harnesses,
  the tier-to-model mapping, and the agent limit belong to the runner.
- Whether a runner's usage-limit brake should lift itself at the reset time the harness reported, now that the brake is
  narrow enough that lifting it wrongly costs one runner's night rather than the machine's.
- Whether the runner panel on the machine becomes a host panel with a view per runner, or each runner keeps a panel of
  its own.
