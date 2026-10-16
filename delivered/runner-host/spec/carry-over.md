# Carry-over contract

Nothing deployed today may need re-enrolling, lose a chunk, or change id. Every existing runner becomes a host of one at
graduation, and a host gathers further runners by ingesting them. Blizzard ships no command that converts a runner's
configuration: converting a runner's file is a step of deploying the release that introduces the host, carried out by
hand (§Graduation).

## How the host reaches a running fleet

The fleet that builds the host is also the first to run it, and each landing is deployed to it before the next begins.
The hub redeploys itself from `master`; the runners are redeployed after every landing by a standard pass — install the
new release, migrate each store while its daemon is still running, then restart — which, in this workspace's fleet, a
worker carries out from inside one of the runners it restarts. Every landing of this epic but one is shaped to survive
that pass, and the order they land in is what makes that possible:

1. **The hub's side**, all of it additive: hosts, the host credential and its lane, bindings, and the narrowed peek are
   things no runner is obliged to use.
2. **The store's additive revision** ([store.md](./store.md) §Two revisions), under runners still reading
   `blizzard-runner.toml`, so the largest schema change of the epic is proven on the live stores before anything else
   moves.
3. **The host release**: `blizzard-host.toml` and the refusal of a bare `blizzard-runner.toml`, the tightening revision,
   and the daemon composed as a host of its runners. This is the exception. Its deploy stops each runner, converts its
   directory, and migrates its store while it is down, and a worker cannot stop the runner it runs inside and live to
   finish the job. So the host release is delivered by hand and deployed in one attended session with the fleet drained
   (§Graduation), by an operator or an agent working outside the fleet.
4. **Everything after**, on graduated hosts and by the standard pass again, each store change additive once more: the
   panel's runner views, routing across bindings and the shared pool, subscriptions sampled on the host, registration
   and added runners, and ingest.
5. **Gathering hosts into one** (§The dogfood fleet) is an attended operation after the last landing, not a landing of
   its own.

A standard pass that reaches a runner directory before it has graduated stops before it restarts anything:
`blizzard runner migrate` refuses a directory holding `blizzard-runner.toml` and no `blizzard-host.toml`, naming
§Graduation and changing nothing.

## Hub and runners on different clocks

The hosted hub follows `master` on its own timer — minutes after a landing, or not at all while its image is pinned back
— so a runner may run a newer release than its hub for as long as that lasts, and an older one just as often. Every host
field of the wire is additive (`bzh:fleet-wire-additive`), and a hosted runner keeps sending everything a hub without
hosts reads: its `workspace_id` and `env_capacity`, which name its first binding's workspace and that workspace's size,
and the `projects` its bindings declare, beside the `bindings` and `subscription` a newer hub reads instead
([hub.md](./hub.md) §Registration).

A runner whose host is not registered registers, reports its roster and samples, and signs in to its panel exactly as a
host-less runner does today, and the board shows it standing alone. A registered host whose hub answers its report with
a not-found — a hub from before hosts — keeps every runner working, holds its own facts in its outbox until a hub takes
them, and offers no panel sign-in until one does.

## Graduation

A release with the host refuses to start from a directory holding `blizzard-runner.toml` and no `blizzard-host.toml`,
naming this procedure. The procedure is short and mechanical, an agent carries it out as readily as a person, and it is
run once, in the attended session that delivers the host release, for every runner on the machine together: runners
sharing one installation cannot graduate one at a time, since the release they share refuses whichever is left.

1. **Rehearse.** With the new release installed apart from the one the runners run, convert a copy of each runner
   directory, migrate the copy, and check it (step 6). A failure here costs nothing.
2. **Drain.** Pause every runner at the hub, so nothing new is claimed, and let any chunk that has already landed finish
   its redeploy pass. Close every takeover, whose session holds an address the restart retires. Work in the middle of a
   node needs no draining, since the restart resumes it in place.
3. **Stop** every runner, and copy each directory's store files — `data/runner.db` with its `-wal` and `-shm` siblings —
   its `blizzard-runner.toml`, and its `.env` to a dated backup.
4. **Convert the file.** The runner's directory becomes the host directory, and `blizzard-runner.toml` becomes
   `blizzard-host.toml` by the key placement in [configuration.md](./configuration.md) §Where each key lives. The host
   keys stay at the top. `workspace_*` and the prompt keys become one `[[workspace]]`, whose `id` is the old
   `workspace_id`. Everything else becomes one `[[runner]]`:
   - its `id` is the hub id its store records as its identity — the id `hub runner list` shows beside its name — and its
     `name` is the old `name`, or the legacy `runner_id` key, which held a name;
   - its `projects` are bindings to that workspace with `agents` set to the old `max_agents` or, where the file had no
     `projects` key, a binding for each project the hub last answered it serves (`hub_served_projects`,
     `runner/store/schema.py:350`);
   - its `subscription` is its one declared slug. A file declaring several moves every declaration to the host and has
     the runner name the plan its enabled harness spends. A file declaring none sampled the `anthropic` plan synthesized
     from `[external_subscription_usage]` (`runner/config.py:779`); the host now declares that plan explicitly, with the
     legacy table's interval and credentials path, and the runner names it;
   - every path key keeps its value. The files the directory already holds — the harness bundle, worker settings, and
     scratch directories — stay where they are, so no stored path goes stale, an open elicitation's output file
     (`in_flight_elicitations.output_path`) among them; only a runner added later takes `runners/<name>/`.

   The token in `.env` is renamed `BZ_HUB_TOKEN_<NAME>`, and the old file is kept beside the new one as
   `blizzard-runner.toml.graduated`.
5. **Migrate the store.** `blizzard runner migrate` adopts `data/runner.db` as `data/host.db` when no host store exists
   — only while no daemon answers on the directory's `runner.sock`, folding the write-ahead log into the database first
   and moving its siblings with it — and runs the tightening revision ([store.md](./store.md) §Two revisions).
6. **Check.** `blizzard runner host --check` loads the file, verifies the store's revision, and resolves each runner's
   token to its `id`, starting nothing ([process.md](./process.md) §CLI). It must pass before anything starts.
7. **Start** the host, confirm each runner's status and the host's `/api/health`, and resume the runners at the hub.

The result is a host of one, unregistered at the hub, running exactly the runner that was there, under the same id and
token, with every chunk it held. Held `env_bindings` rows are completed on its first tick (§Backfilling held bindings),
and workers alive before the restart are respawned by resume with the host's address ([process.md](./process.md)
§Listeners and the panel).

Going back is the backup: stop the host, restore the copied files over the converted ones, reinstall the release the
runners ran before, and start them. Nothing at the hub needs undoing, since a host of one is unknown to it.

## Registering a host

An unregistered host may run exactly one runner. Running more — by ingest or by `blizzard runner add` — needs a host the
hub knows: `hub host add <name> --adopt <runner id>…` prints the host's id and token; the token goes into the host
directory's `.env` as `BZ_HOST_TOKEN`; and on its next start the host reports itself and records its identity
([process.md](./process.md) §One tenant). From then on it is the panel's sign-in client and reports its subscriptions on
its own lane. A registered host refuses to run a runner the hub does not hold under it.

An adoption is undone by releasing the runner from its host (`hub host release`, [hub.md](./hub.md) §Operator verbs),
never by retiring the host, which retires every runner under it.

## Ingest

`blizzard runner ingest <source dir> --runner <name|id> [--workspace-id <id>]` moves one runner's data from another
store into this host's. The source is another host's directory or a runner directory not yet graduated; either way
ingest reads a current-schema copy of its store.

1. **Refuse** unless this host is stopped (its lock is free); the source is stopped (its lock is free, and no
   `runner.sock` answers in it); this host is registered and the hub holds the runner under it; the runner's identity in
   the source names this host's tenant; and the runner holds no open elicitation or takeover. A runner whose workspace
   `id` this host already gives to a different root or set of environments is refused unless `--workspace-id` names a
   new one. A host that has not yet recorded its identity reads it from the hub under its token (`GET /api/fleet/host`)
   and records it, so a freshly registered host needs no start before its first ingest.
2. **Copy the source store** to a temporary file and migrate the copy to the current schema. The source itself is never
   written by this step.
3. **Copy the runner's rows** from the copy into this host's store in one transaction, table by table in the order the
   reference declarations require, re-numbering every integer surrogate key and every column that refers to one
   ([store.md](./store.md) §Integer keys and references). Outbox rows keep their `seq`s, which are the runner's own, and
   its outbox counters come with them, so every fact still lands above the runner's high-water mark at the hub. Under
   `--workspace-id`, the copied rows' `workspace_id` and the runner's prompt override take the new id. Host-scoped rows
   merge: subscription samples and renewal records by slug, keeping both histories; graph artifacts by their key, which
   names the same immutable rows in both stores; and a workspace-prompt override is copied when this host has none for
   that workspace, while two different overrides for one workspace refuse the ingest, naming both. The same transaction
   records the ingest: the runner, the source, and the source's last liveness stamp.
4. **Add the runner's configuration**, each edit skipped when it is already made. The runner's `[[runner]]` entry is
   appended to `blizzard-host.toml`, with every path key that points into the source rewritten to a copy of its file
   under this host's `runners/<name>/`; its workspace is appended unless this host already declares one with that root
   and environments; and its subscription unless this host already declares that slug. Its token moves from the source's
   `.env` to this host's, and that move is what fences the source: without its token, the source cannot run the runner
   again.
5. **Remove the runner from the source.** In a separate transaction, delete its rows from the source store. A source
   left with no runner is moved aside whole under a timestamped name.

Steps 3 to 5 touch separate files and cannot share one transaction, so ingest resumes rather than repeats. A runner
already in this host's store, with an ingest recorded from this source whose liveness stamp has not moved since, is an
ingest interrupted part-way: re-running it completes steps 4 and 5 and copies nothing again. If the source has run since
the copy, the copy may be stale, and ingest refuses, naming both stores.

## Backfilling held bindings

`env_bindings` gains `workspace_id` and `project_id` ([workspaces.md](./workspaces.md) §Routing a claim), and rows
written before the additive revision have neither. Neither a migration nor ingest can read a host's configuration or
call the hub, so each runner completes its held rows — every row with no later release, leased or not — at start, before
its first FILL:

- A row whose `workspace_id` is empty, or names a workspace the host does not declare, takes the workspace whose root
  and environments match its `workdir` — never by environment name alone, since two workspaces on different roots may
  both name `r1`. A row that matches none keeps its lease, counts against every binding of its runner until it is
  released, and is named in the runner's status.
- A row whose `project_id` is empty takes the project the hub names for its chunk through the fleet chunk view
  (`GET /api/fleet/chunks/{chunk_id}`, `hub/api/fleet.py:268`). Until then it counts against every binding on its
  workspace ([workspaces.md](./workspaces.md) §Parallel agents).

So the backfill can delay a claim but never over-admit one.

## The dogfood fleet

This workspace's two runners, `r-claude` in `../runner` and `r-chatgpt` in `../runner-opencode`, share one installation
and one checkout, and both declare `workspace_id = "workspace-local"`.

1. **Graduate both** in the session that delivers the host release (§Graduation). Each becomes an unregistered host of
   one, in its own directory, under its own unit and port, as before.
2. **Gather them** in a second attended session, once registration and ingest have landed, with both runners paused:
   1. `hub host add <name> --adopt rn_01M3MY1NGQR8WE1ZBRKNREFNE4 rn_01M3JK4RX4BQJ74W7T3CK4NQ1B`, placing the host token
      in `../runner/.env` as `BZ_HOST_TOKEN`.
   2. Stop both, and back up both directories as graduation does.
   3. From `../runner`, `blizzard runner ingest ../runner-opencode --runner r-chatgpt --workspace-id blizzard-openai`,
      so the two workspaces on one checkout — `r1`–`r6` and `oce1`–`oce4` — keep ids of their own.
   4. `blizzard runner host --check`, then start `../runner` as the one host — two workspaces, two runners, two
      subscriptions — confirm each runner's status and trace export, and resume both.
   5. Fold the second systemd unit's settings into the first and retire it, and rewrite the workspace's instance
      documents for one host, one unit, and one port.

Going back from the gathering is the backup again, with both runners released from the host at the hub.
