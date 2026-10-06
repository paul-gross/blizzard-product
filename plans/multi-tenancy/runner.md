# Plan — `epic:multi-tenancy`, runner slice

The hub slice settles a runner's tenant without asking the runner anything. The token a runner holds resolves to one
registration in one tenant, and a runner built before tenancy keeps claiming and reporting as if nothing had changed.
What the hub cannot settle is what the runner remembers, and what the runner does on the operator's behalf. A runner
keeps a local store of its leases, the events it has not yet delivered, and the chunks it is still driving, and on a
machine where the hub needs no sign-in it adds itself at the hub when it is set up. Neither knows that the hub now holds
more than one world.

This slice makes a runner know whose world it works in, and refuse to mix two.

It is built on runners that the hub already identifies by an id it minted rather than by a name the runner chose, so
that a runner learns its id when it registers, its name is only a label, and no two runners anywhere share an id.
Everything below assumes that ground and would be harder to build on any other.

## One runner, one tenant, for life

A runner process never serves two tenants, and a runner never moves between them. The tenant a runner serves is the one
it was added in, and its local store belongs to that runner alone. Moving a machine's work to another tenant means
adding a new runner there, with a fresh store, rather than re-pointing the old one. That keeps tenancy out of the
runner's store entirely: one store, one runner, one tenant, so no table needs a tenant column and no query needs a
filter.

## What to build

- **A runner refuses a store that is not its own.** The store keeps the id of the runner that wrote it, and beside it
  the tenant the hub answered with, for display. At its first contact with the hub after each start, the runner checks
  that its token still resolves to that id. When the token resolves to a different runner, the runner refuses to work:
  it delivers no buffered event, claims nothing, and drives no lease further. Its status and log name the id the store
  holds, the id the token resolved to, and the command that clears the store. It keeps running so its status stays
  readable, and checks again at each contact, so restoring the right token resumes it with nothing lost. Before that
  first contact it does only what it already does while the hub is unreachable, so an outage never reads as a mismatch.
  The check is on the runner's id, not its tenant: ids are unique across the hub and a runner never changes tenant, so a
  matching id settles the tenant too. Without it, a runner handed another runner's token would deliver yesterday's
  events as a runner that never produced them, and keep working chunks it does not own.
- **The runner never discards its store.** A mismatch most often means a wrong token or a hub URL pointing somewhere
  else, and then the undelivered events and in-flight leases in the store belong to the hub the runner should have
  reached; throwing them away would lose work that hub is still waiting on. Clearing is the operator's deliberate act:
  `blizzard runner store reset <dir>` moves the store aside under a timestamped name and never deletes it.
- **Throwaway environments opt in.** `runner init` re-adds a runner whose token the hub does not know only under
  `--allow-readd`, and when it does, it also sets the old store aside, as after a feature environment's hub data is
  reset. A plain restart reuses the token and keeps the store. The feature-environment runner service, blizzard-mock's
  fleet, and the end-to-end and test harnesses that recycle directories pass the flag; a production runner never does,
  so a runner pointed at the wrong hub stops at `init`, or refuses to work, rather than starting over.
- **Setting up a runner names its tenant.** `runner init --tenant <tenant_id>` adds the runner in the tenant that id
  names; a tenant is named by its id, never its name. Without the flag, init resolves the tenant as any request does:
  the operator's only membership, or the hub's only tenant on a hub without sign-in — which, on a carried-over hub, is
  always. When that leaves a choice, init stops, adds nothing, and lists the tenants the operator may enter, ids beside
  names. When `.env` already holds a token whose runner belongs to a different tenant than `--tenant` names, init stops
  and adds nothing too, naming both tenants: a runner never moves between tenants, so the operator either drops the flag
  or sets up a new runner in a fresh directory. A test that brings up a runner in a tenant of its own does it with the
  same command the operator uses.
- **Sign-in needs no tenant check of its own.** A person signs in to a runner's own web surface with a token the hub
  issues for that runner, carrying the person's role in the runner's tenant. The hub issues it only to a member of that
  tenant, and the runner accepts it only when it names the runner's own id, which belongs to exactly one tenant — so the
  role it grants is always a role in the world the runner serves, with nothing further for the runner to compare.
- **A runner names its tenant wherever it names itself.** Its status, the command it suggests for resuming it, and the
  attributes on its traces all carry the tenant beside its id and name — the tenant its registration answers with, kept
  in its store for display — so an operator whose runners serve several tenants can tell them apart at a glance and in a
  trace search.
- **Existing runners carried over.** A runner whose store was written before this slice adopts the tenant its token
  resolves to on its first start after the upgrade — on a carried-over hub, `default` — with no configuration change and
  no stranded chunk.

## What it leaves to others

Running many runners side by side on one machine — their ports, workspace roots, and harness credentials — belongs to
`epic:runner-host`, which gives each runner a host to share instead of a desk to fight over. Once a host runs several
runners, the binding above belongs to each runner under it, not to the host. Which projects a runner serves inside its
tenant belongs to `epic:projects`' runner slice. What the hub adds to its wire for this slice is specified with the hub
slice, in its [API contract](./hub/spec/api.md).

The runner-side tests `epic:test-shared-service` wants need only what this slice and the hub slice give them: each test
adds a runner in its own tenant, starts it on a store of its own, and cannot reach another test's world, because the hub
refuses the crossing and the runner refuses the confusion.
