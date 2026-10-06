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

- **The store remembers whose it is.** The first time a runner registers, it records the id and tenant the hub answers
  with. On every start after that it checks them, and a token that resolves to a different runner refuses to start,
  naming what the store holds and what the token resolved to, before a single buffered event is replayed or a lease is
  driven. Without that check, a runner handed another runner's token would replay yesterday's undelivered events into a
  world that never produced them, and keep working chunks that world has never heard of. Clearing the store is the
  operator's deliberate act, never the runner's quiet one.
- **Setting up a runner names its tenant.** Where `runner init` adds the runner at its hub, it adds it in the tenant the
  operator names, and needs no name only when there is one tenant to choose from — which, on a carried-over hub, is
  always. A test that brings up a runner in a tenant of its own does it with the same command the operator uses.
- **Sign-in checks the tenant too.** A person signs in to a runner's own web surface with a token the hub issues for
  that runner, carrying the person's role in the runner's tenant and the tenant itself. The runner accepts it only when
  the tenant matches its own, so the role it grants is always a role in the world it serves.
- **A runner names its tenant wherever it names itself.** Its status, the command it suggests for resuming it, and the
  attributes on its traces all carry the tenant beside its id and name, so an operator whose runners serve several
  tenants can tell them apart at a glance and in a trace search.
- **Existing runners carried over.** A runner whose store was written before this slice adopts the tenant its token
  resolves to on its first start after the upgrade — on a carried-over hub, `default` — with no configuration change and
  no stranded chunk.

## What it leaves to others

Running many runners side by side on one machine — their ports, workspace roots, and harness credentials — belongs to
`epic:runner-host`, which gives each runner a host to share instead of a desk to fight over. Once a host runs several
runners, the binding above belongs to each runner under it, not to the host. Which projects a runner serves inside its
tenant belongs to `epic:projects`' runner slice. What the hub adds to its wire and its sign-in tokens for this slice is
specified with the hub slice, in its [API contract](./hub/spec/api.md).

The runner-side tests `epic:test-shared-service` wants need only what this slice and the hub slice give them: each test
adds a runner in its own tenant, starts it on a store of its own, and cannot reach another test's world, because the hub
refuses the crossing and the runner refuses the confusion.

## Open questions

- **Refuse or set aside.** A mismatched store could instead be moved aside and a fresh one started, which suits a test
  harness that recycles directories. The leaning is to refuse, since a quiet fresh start on a production runner strands
  every chunk it was driving.
