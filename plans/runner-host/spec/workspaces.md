# Workspaces contract

## Bindings

A runner's `projects` bindings ([configuration.md](./configuration.md) §Projects) are the `[runner, project,
workspace]`
relation, unique on `[runner, project]`, each carrying its `agents`. A workspace may appear in any number of bindings,
across any number of runners; a project appears at most once per runner.

At registration the runner sends its bindings (`bindings`, [hub.md](./hub.md) §Registration), from which the hub derives
the projects it serves, and its host reports the workspaces they name. The hub's answer (`hub_served_projects`,
`runner/store/schema.py:348`) resolves each binding's project entry to a project id, and the runner keeps a routing map,
project id → binding, rebuilt from that answer and its bindings at every registration. An entry that resolves to nothing
routes nothing. Two entries that resolve to one project are refused ([configuration.md](./configuration.md)
§Validation).

## Routing a claim

A runner peeks per binding, not once for everything it serves. The hub's matched peek (`POST /api/fleet/queue/peek`,
`wire/queue.py:37`) already answers with the first eligible entry in queue order among the runner's served projects;
`QueuePeekRequest` gains, additively under `bzh:fleet-wire-additive`, an optional `project` that narrows that same
answer to one project of the runner's declaration. Ordering, eligibility, capabilities, and queue policy are the hub's,
unchanged. A project outside the runner's declaration answers with nothing, and a request without `project` behaves as
today.

In FILL (`runner/lifecycle/claim.py:120`, `runner/loop/steps.py:435`) the runner walks its bindings in configuration
order. For each binding with a free agent (§Parallel agents) it peeks with that binding's project, once per free agent,
and stops for that binding at the first empty answer. A claimed entry acquires an environment from the binding's
workspace and claims with `POST /api/fleet/routes` carrying that workspace's id. A binding with no free agent sends no
peek at all, so a full project never stands in front of another project's work.

`QueuePeekEntry` carries the chunk's `project_id` (`wire/queue.py:26`). An entry whose project is not the binding's —
possible only against a hub too old to honor `project` — is passed over and not claimed.

The workspace and binding a chunk was claimed through are fixed for the chunk's tenure: `env_bindings`
(`runner/store/schema.py:48`) gains `workspace_id` and `project_id`, so recovery, release, and resume read them from the
row and never re-route. A binding changed in configuration affects only the next claim.

## Parallel agents

`max_agents` leaves the runner. FILL's slot count becomes a count per binding: a binding's open leases are those whose
`env_bindings` row names its project, and its free slots are its `agents` less that count. A held row whose project is
not yet known ([carry-over.md](./carry-over.md) §What the host does on first start) counts against every binding on its
workspace, so an unknown row can only make a binding claim less, never more. The runner has no cap of its own beyond the
sum of its bindings'.

`agents` is independent of the workspace's environments. A binding allowed more agents than its workspace has free
environments claims only as environments free, since a failed acquire leaves the chunk waiting as it does today
(`claim.py:195`); a binding allowed fewer leaves the rest of the pool for other bindings.

## The shared pool

A workspace's environments are shared by every binding to it, whichever runner holds the binding. Occupancy stays the
fact it is today — an `env_bindings` row with no later `binding_releases` — now in the one host store, where every
runner's rows sit side by side. A workspace's held set is one read across all of them, a named `HostStores` member
([store.md](./store.md) §Scoped access). The workspace seam already takes the held set as an argument
(`IWorkspaceProvider.acquire(chunk_id, count, held_ids)`, `runner/environments/provider.py:66`), so a provider needs no
change: the runner reads the held set, has the provider choose, and records its binding in one `BEGIN IMMEDIATE`
transaction, so two runners can never bind one environment.

```text
acquire(workspace, runner, chunk):
  BEGIN IMMEDIATE
    held = host_stores.held_environment_ids(workspace)        -- every runner's held rows
    env  = workspace.provider.acquire(chunk, 1, held)
    runner_stores.environments.bind(chunk, workspace, project, env)
  COMMIT
```

The provider's own preparation of the environment — fetching, resetting, provisioning — runs after the commit, outside
the transaction, as it does today; the binding already reserves the environment. Two workspaces on one root are two
pools: each acquires only from the environments it names.

## Crash recovery

Occupancy is durable in the host store, so a host restart recovers it with no further record: each runner's
`InterruptedClaims.reconcile` (`claim.py:252`) walks its own held bindings as it does today, and the next acquire on a
workspace sees every binding any runner still holds. A runner that is refusing to work — a mismatched identity, a
foreign tenant — keeps its held rows, so its environments are never handed to a neighbour while its chunks still hold
them. That an environment is held by at most one binding across the host is asserted by the invariant checker
([store.md](./store.md) §Invariants).

## Workspace prompts

The standing prompt is the workspace's: `Spawner` (`runner/lifecycle/spawn.py:595`) reads the prompt of the workspace
the lease's `env_bindings` row names. The live override (`GET/PUT/DELETE /api/workspace-prompt`,
`api/workspace_prompt.py:37`) moves to the host's routes as `/api/host/workspaces/<id>/prompt`, and its row becomes
host-scoped, keyed by workspace id ([store.md](./store.md)). A runner's own `runner_prompt` stays the runner's.

## What a workspace holds

Blizzard does not check, at binding or at claim, that a workspace holds the repositories its projects link. What a
workspace's environments contain, and whether a repository is cloned when a chunk first needs it, belongs to the
provider behind the workspace seam, not to the host.
