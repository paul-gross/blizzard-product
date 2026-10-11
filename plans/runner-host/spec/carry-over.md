# Carry-over contract

Nothing deployed today may need re-enrolling, lose a chunk, or change id. Blizzard ships no command for the move: it is
a short procedure over files and two hub verbs, written down here and in the runner's documentation, and an agent
carries it out as readily as a person.

## The hub first

The hosted hub redeploys ahead of the runners, so it accepts a runner with no host for as long as such a runner talks to
it. A host-less runner registers, reports its roster and samples, and signs in to its own panel exactly as today; the
host fields of the wire are additive (`bzh:fleet-wire-additive`), and the board shows it standing alone. Nothing about
the fleet changes until its operator moves a machine.

## Moving a machine under a host

1. **Add the host and adopt the runners.** `hub host add <name> --adopt <runner id>…` in the runners' tenant prints the
   host's id and token. Adoption only records `host_id` on each runner, so it is safe while they run.
2. **Stop the runners.** `blizzard runner host` refuses a directory still served by a live daemon (`cli/daemon.py:176`).
3. **Write `blizzard-host.toml`** in a new host directory, with the token in that directory's `.env` as `BZ_HOST_TOKEN`:
   - the host-level keys — `hub_url`, listener, `[auth]`, `[tracing]`, `[transcripts]` — from the runners' files, which
     must agree on them;
   - one `[[workspace]]` per runner, naming exactly that runner's `workspace_envs` under its `workspace_root`, so each
     keeps the pool it had; an operator who wants the pools merged declares one workspace naming them all;
   - every `[[subscription]]` the runners declared, merged by slug;
   - one `[[runner]]` per runner whose `dir` is its existing directory, so its store, `.env` token, and harness bundle
     stay where they are; its `subscription` the one it samples; its harness, model, security, gate, and cost keys moved
     over unchanged; and its `projects` as bindings to its workspace with `agents` set to its old `max_agents` — for a
     runner whose file had no `projects` key, a binding for each project the hub last answered it serves
     (`hub_served_projects`, `runner/store/schema.py:348`).
4. **Start the host** under one service unit in place of the runners' units, and re-apply any live workspace-prompt
   override through `/api/host/workspaces/<id>/prompt`.

`blizzard runner host` validates the result before it starts anything ([configuration.md](./configuration.md)
§Validation), and refuses a runner the hub does not hold under its host.

## What the host does on first start

Each runner store migrates in place. Held `env_bindings` rows written before the host gain the workspace whose
environments include theirs — environment names are unique within a root, so the match is exact — and the project of
their chunk, as the hub's chunk view reports it during reconcile (`claim.py:252`). Leases, outboxes, and brakes carry on
as they were, so chunks in flight resume where they stopped. The runner store's `workspace_prompt` and
`external_usage_samples` rows are left in place and no longer read. Workers alive before the restart are terminated by
it and respawned by resume with the host's address ([process.md](./process.md) §Listeners and the panel).

## The dogfood fleet

Moving `../runner` and `../runner-opencode` under one host yields the configuration in
[configuration.md](./configuration.md): one checkout declared as two workspaces, `r1`–`r6` and `oce1`–`oce4`; two
runners keeping `rn_01M3MY1NGQR8WE1ZBRKNREFNE4` and `rn_01M3JK4RX4BQJ74W7T3CK4NQ1B`; two subscriptions; and one systemd
unit where there were two.
