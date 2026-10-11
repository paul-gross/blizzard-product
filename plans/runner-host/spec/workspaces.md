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

`QueuePeekEntry` already carries the chunk's `project_id` (`wire/queue.py:26`). In FILL
(`runner/lifecycle/claim.py:120`) the runner looks the entry's project up in its routing map before anything else:

1. **Bound, with a free agent.** The entry's binding has fewer open leases than its `agents`. Acquire, bind, and claim
   proceed against the binding's workspace, and `POST /api/fleet/routes` carries that workspace's id.
2. **Bound, and full.** Every agent the binding allows is busy. The entry is one the runner cannot take now, handled
   under its `[queue]` policy exactly as an entry it lacks a slot for is handled today.
3. **Unbound.** The hub offered a chunk of a project the runner does not bind — a stale declaration, or a binding just
   removed. The runner passes over it and claims nothing for it.

The workspace and binding a chunk was claimed through are fixed for the chunk's tenure: `env_bindings`
(`runner/store/schema.py:48`) gains `workspace_id` and `project_id`, so recovery, release, and resume read them from the
row and never re-route. A binding changed in configuration affects only the next claim.

## Parallel agents

`max_agents` leaves the runner. FILL's slot count (`runner/loop/steps.py:435`) becomes a count per binding: a binding's
open leases are those whose `env_bindings` row names its project, and its free slots are its `agents` less that count.
The runner has no cap of its own beyond the sum of its bindings'.

`agents` is independent of the workspace's environments. A binding allowed more agents than its workspace has free
environments claims only as environments free, since a failed acquire leaves the chunk waiting as it does today
(`claim.py:195`); a binding allowed fewer leaves the rest of the pool for other bindings.

## The shared pool

A workspace's environments are shared by every binding to it, whichever runner holds the binding. Occupancy stays where
it is today — in each runner's own store, as `env_bindings` with no later `binding_releases` — and the host derives a
workspace's held set as the union of every runner's held environments for that workspace. The workspace seam already
takes the held set as an argument (`IWorkspaceProvider.acquire(chunk_id, count, held_ids)`,
`runner/environments/provider.py:66`), so a provider needs no change: the host calls `acquire` with the host-wide held
set, under the workspace's own lock, and the runner records its binding before the lock is released.

```text
acquire(workspace, chunk):
  with workspace.lock:
    held = ⋃ runner.stores.environments.held_environment_ids(workspace)  for every runner bound to workspace
    env  = workspace.provider.acquire(chunk, 1, held)
    runner.stores.environments.bind(chunk, workspace, project, env)
```

Two workspaces on one root are two pools: each acquires only from the environments it names.

## Crash recovery

Because occupancy lives in runner stores, a host restart recovers it with no host record: each runner's
`InterruptedClaims.reconcile` (`claim.py:252`) walks its own held bindings as it does today, and the host's first
acquire on a workspace sees every binding any runner still holds. A runner that is refusing to work — a mismatched
identity, a foreign tenant — still contributes its held bindings to the union, so a halted runner's environments are
never handed to a neighbour while its chunks still hold them.

## Workspace prompts

The standing prompt is the workspace's: `Spawner` (`runner/lifecycle/spawn.py:595`) reads the prompt of the workspace
the lease's `env_bindings` row names. The live override (`GET/PUT/DELETE /api/workspace-prompt`,
`api/workspace_prompt.py:37`) moves to the host's routes as `/api/host/workspaces/<id>/prompt`, and its row moves from
each runner's `workspace_prompt` table to the host store's ([carry-over.md](./carry-over.md)). A runner's own
`runner_prompt` stays the runner's.

## What a workspace holds

Blizzard does not check, at binding or at claim, that a workspace holds the repositories its projects link. What a
workspace's environments contain, and whether a repository is cloned when a chunk first needs it, belongs to the
provider behind the workspace seam, not to the host.
