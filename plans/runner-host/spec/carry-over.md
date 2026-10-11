# Carry-over contract

Nothing deployed today may need re-enrolling, lose a chunk, or change id. Every existing runner becomes a host of one
when it is next deployed, and a host gathers further runners by ingesting them. Blizzard ships no command that rewrites
configuration: converting a runner's file is part of deploying the release that introduces the host, carried out by
whoever deploys it — in this fleet, the agent that redeploys the runners.

## The hub first

The hosted hub redeploys ahead of the runners, so it accepts a runner with no host for as long as such a runner talks to
it. Every host field of the wire is additive (`bzh:fleet-wire-additive`). A runner whose host is not registered at the
hub registers, reports its roster and samples, and signs in to its panel exactly as a host-less runner does today, and
the board shows it standing alone.

## Graduation

A release with the host refuses to start from a directory holding `blizzard-runner.toml` and no `blizzard-host.toml`,
naming this procedure. Deploying it to a runner is:

1. **Stop the runner**, as every redeploy does.
2. **Convert the file.** The runner's directory becomes the host directory. `blizzard-runner.toml` becomes
   `blizzard-host.toml` by the key placement in [configuration.md](./configuration.md) §Where each key lives: the host
   keys stay at the top, `workspace_*` and the prompt keys become one `[[workspace]]`, and everything else becomes one
   `[[runner]]` whose `name` is the old `name` (or legacy `runner_id`), whose `subscription` is its one declared slug,
   and whose `projects` are bindings to that workspace with `agents` set to the old `max_agents` — or, where the file
   had no `projects` key, a binding for each project the hub last answered it serves (`hub_served_projects`,
   `runner/store/schema.py:348`). The token in `.env` is renamed `BZ_HUB_TOKEN_<NAME>`, and the runner's harness bundle
   and scratch directories move under `runners/<name>/`. The old file is kept beside the new one as
   `blizzard-runner.toml.graduated`.
3. **Migrate the store.** `blizzard runner migrate`, which every store change already requires (`runner/runtime.py:198`
   checks the revision at start and refuses a stale one), renames `data/runner.db` to `data/host.db` when no host store
   exists and runs the chain forward: the graduating revision adds `runner_id` to every runner-scoped table and fills it
   from the store's own `runner_identity` row ([store.md](./store.md)). A store that holds runner-scoped rows but no
   identity is refused rather than guessed at.
4. **Start the host.**

The result is a host of one, unregistered at the hub, running exactly the runner that was there, under the same id and
token, with every chunk it held. Held `env_bindings` rows are completed on the first tick (§Backfilling held bindings).

## Registering a host

An unregistered host may run exactly one runner. Running more — by ingest or by `blizzard runner add` — needs a host the
hub knows: `hub host add <name> --adopt <runner id>…` prints the host's id and token; the token goes into the host
directory's `.env` as `BZ_HOST_TOKEN`; and on its next start the host reports itself and records its identity
([process.md](./process.md) §One tenant). From then on it is the panel's sign-in client and reports its subscriptions on
its own lane. A registered host refuses to run a runner the hub does not hold under it.

## Ingest

`blizzard runner host ingest <source dir> --runner <name|id>` moves one runner's data from another store into this
host's. The source is another host's directory or a runner directory not yet graduated; either way ingest reads a
current-schema copy of its store.

1. **Refuse** unless this host is stopped (its lock is free), the source is stopped (its lock is free, and no
   `runner.sock` answers in it), this host is registered, the hub holds the runner under this host, the runner's
   identity in the source names this host's tenant, and the runner is not already in this host's store.
2. **Copy the source store** to a temporary file and migrate the copy to the current schema. The source itself is never
   written by this step.
3. **Copy the runner's rows** from the copy into this host's store in one transaction, table by table in the order the
   reference declarations require, re-numbering every integer surrogate key and every column that refers to one
   ([store.md](./store.md) §Integer keys and references). `seq` columns of the outbox tables are kept as they are.
   Host-scoped rows merge: subscription samples and renewal records by slug, keeping both histories; a workspace-prompt
   override is copied when this host has none for that workspace, and two different overrides for the same workspace
   refuse the ingest, naming both.
4. **Add the runner's configuration.** The runner's `[[runner]]` entry, its workspace if this host does not already
   declare one with that root and environments, and its subscription if this host does not already declare that slug are
   appended to `blizzard-host.toml`; its token and harness bundle move under this host.
5. **Remove the runner from the source.** In a separate transaction, delete its rows from the source store. A source
   left with no runner is moved aside whole under a timestamped name.

Steps 3 and 5 are separate transactions in separate files. A crash between them leaves the runner in both stores; that
is accepted. The hub holds the runner under this host, so the source host refuses to run it, and re-running ingest finds
the runner already present, skips steps 2 to 4, and completes step 5.

## Backfilling held bindings

`env_bindings` gains `workspace_id` and `project_id` ([workspaces.md](./workspaces.md) §Routing a claim). Rows written
before the host have neither. The graduating migration and ingest fill `workspace_id` from the row's `workdir`, matched
against each workspace's root and environments — never by environment name alone, since two workspaces on different
roots may both name `r1`. Neither may call the hub, so `project_id` stays null. On each runner's first tick under the
host, before FILL, the runner asks the hub for the project of every held row whose `project_id` is null — leased or not
— through the fleet chunk view (`GET /api/fleet/chunks/{chunk_id}`, `hub/api/fleet.py:267`) and records it. Until then
the row counts against every binding on its workspace ([workspaces.md](./workspaces.md) §Parallel agents), so the
backfill can delay a claim but never over-admit one.

Workers alive before the restart are terminated by it and respawned by resume with the host's address
([process.md](./process.md) §Listeners and the panel).

## The dogfood fleet

1. Deploy the release to both runners: `../runner` and `../runner-opencode` each graduate into a host of one, running as
   they do today.
2. `hub host add <name> --adopt rn_01M3MY1NGQR8WE1ZBRKNREFNE4 rn_01M3JK4RX4BQJ74W7T3CK4NQ1B`, and place the host token
   in `../runner/.env`.
3. Stop both, and from `../runner` run `blizzard runner host ingest ../runner-opencode --runner r-chatgpt`.
4. Start `../runner` as the one host: two workspaces on one checkout, `r1`–`r6` and `oce1`–`oce4`; two runners; two
   subscriptions. Retire the second systemd unit.
