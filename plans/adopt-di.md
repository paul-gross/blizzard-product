---
epic: adopt-di
refinement: refined
status: horizon
---

# Plan — `epic:adopt-di`

Most of blizzard is written by agents meeting a module for the first time (`persona:fleet-agent`), and an agent adding a
service to the hub learns how it is wired by editing the wiring. It adds a field to a frozen bag of services, adds a
constructor call to a builder some 370 lines long, and does both inside a composition module of more than 1,500 lines
that every other agent adding a service is editing the same week. The route that uses the new service then takes the
whole bag, so its signature says nothing about what it touches, and a reviewer reading it cannot tell which state a
change can reach.

The bags have grown faster than anyone was watching. Measured on `master` at `d2e3e916e`, the hub's `TenantServices`
holds 87 members and its `HubServices` 37, where in mid-July a single bag held 11. Some 220 route handlers take the
whole tenant bag through one dependency, and four of its members are read by no route at all — in a flat list of 87,
nobody notices when one stops being used. A bag that every caller reaches into is a service locator by another name: it
makes depending on everything the cheapest dependency to write. The runner is narrower in use, because
`bzh:narrow-seams` already makes each loop step declare the slice of `LoopContext` it reads, but its process graph is
assembled by hand in the same way, in a `runner/composition.py` of some 1,200 lines.

The rule that produced this shape, `bzh:dependency-injection`, says blizzard has no dependency-injection container, and
records no reason why. This epic replaces the hand-built graphs of the hub, the runner host, and the CLI that drives
them with [dishka](https://github.com/reagento/dishka), a dependency-injection container: a registry that knows how to
build each service from the services it needs, builds each one once for the lifetime it is declared with, hands a route
exactly the services that route asks for, and tears everything down in the reverse of the order it was built. It does
this the way a strangler fig replaces a tree. The new structure grows around the old one, takes over its load one branch
at a time, and the old one is cut away only when nothing rests on it, so at no point does the hub or the runner stop
working while it happens. The hub goes first, because its bags are the larger debt and the graph that proves the shape;
the runner host and the CLI follow it.

## Why dishka

The choice was made on a demonstration. Six candidates rewired the same miniature of the hub's hardest shapes — read and
write ports served by one store, two clocks of the same type, tenants built on first use and closed on demand, a
background pass that runs outside any request — against a behavioral contract none of them was allowed to change:
dishka, dependency-injector, wireup, svcs, injector, and FastAPI's own `Depends` with no library at all. Every candidate
passed. They differed in what they cost.

dishka was the only container that did everything on its public interface and left the domain untouched. A route names
the one service it uses, `FromDishka[FindingExitService]`, rather than the bag. The whole graph between services is
checked when the container is built, before a request is served. A tenant is a child container that shares the process's
engine and broker and keeps its own stores, and closing it tears down that tenant alone. The others fell away for
reasons specific to blizzard: dependency-injector silently ignores its own markers under
`from __future__ import annotations`, which all 46 of blizzard's routing modules use, and gives the type checker almost
nothing to check; wireup reaches a long-lived tenant only through internal interface, on a project with one maintainer;
injector needs more wiring than writing it by hand; svcs is a lookup table rather than a container and checks nothing
before first use.

dishka's weaknesses are known going in, and each one has a guard in the foundation below. It resolves by type, so two
collaborators of the same type can be swapped without a sound: a throttle registered the easy way was handed the virtual
clock instead of the wall clock, with no error. It moves the question "is everything wired?" from the type checker,
which answers it today before anything runs, to a test that builds the container, and it does not check what only a
route asks for. Its child containers take no lock unless given one, and without it thirty-two concurrent first requests
built sixteen stores for one tenant. It ignores constructor defaults, which fifteen of the eighty-two classes in the
hub's bags carry. And it is pre-2.0 and largely one maintainer's project, with a deprecation release already open before
the major.

## The shape of the wiring

The container is dishka's, but the vocabulary is blizzard's. A small registry interface in `foundation/` is all any area
of the code ever sees: a verb to declare a service and the lifetime it lives for, one for a factory, one for a resource
torn down when its lifetime ends, one for a value a lifetime is opened with, and one to alias a port to whatever serves
it. One translation module per process turns those declarations into dishka's providers, and it and the route edge are
the only code that imports dishka. The vocabulary grows only when a real registration needs a verb it lacks, and it
never grows a way to hand dishka's own types through.

Each area wires itself, beside its code. A domain package carries a wiring module whose `register` takes the registry
and, as keyword arguments typed by the package's own ports, the adapters it needs from outside — to register, it must be
given its dependencies. It aliases each port to the adapter it was handed and declares its own services, and that is
all: it never names a concrete adapter, never chooses an adapter's lifetime, and never takes another area's service as
an argument, since those it reaches by type through the graph. Lifetimes belong to the adapter layer. The store, forge,
and harness packages each register their concrete adapters once, with the lifetime each lives for, so an adapter that
serves ports from several areas is still one instance, and the areas aliasing it never know how it is provided.

Each process's composition root shrinks to a list. It reads the configuration, chooses the concrete adapter for every
seam the configuration decides, passes the adapters to each area's `register`, and builds the container with dishka's
strict validation, so a type claimed twice — two owners of one port, two clocks of one type — fails the build. The
root's calls are ordinary typed calls, so pyright still fails a root that forgets an adapter an area demands, or passes
one that does not fit the port. Overrides for tests are the translation module's to offer; the registry the domain sees
has no way to override anything.

Lifetimes are each process's own vocabulary: the hub's process, tenant, and request, and the runner host's host and
runner. The hub's tenant cache stays for the life of the process, holding each tenant's child container, and each runner
is a child container of its host. A collaborator confined to one thread, which no container lifetime describes, stays
hand-managed and enters the graph as a value or a factory.

The CLI uses the same wiring rather than wiring of its own. A command builds a container at the top of its root from the
registrations of the areas its command group serves, and dishka builds a service only when something asks for it, so a
verb that needs a hub client never constructs the store beneath a daemon. That laziness is half of a fast start; the
other half is import, since registering an area imports its classes, so a command group registers only the areas it
serves and a worker verb never imports a daemon's. The carrier a test hands a command today becomes an override of the
command's container.

## How it lands

The epic moves through four phases, and only the first changes how anything is built all at once. Each later landing is
small, mechanical, and behavior-preserving.

### Foundation: one container per process, holding today's graph whole

Each process gains its container and its composition root, and the container starts out knowing nothing it has to build
itself. Today's builders become one more registration, a legacy module that answers for `HubServices`, `TenantServices`,
and the stores beneath them by asking those builders for the instances they already make; the runner host's answers for
its host stores, each runner's process collaborators, and each runner's `LoopContext` the same way. A tenant becomes a
child container held in the hub's existing tenant cache, and each runner a child container under its host, so the
lifetimes the processes already have — the process, the tenant, the runner — are the container's scopes from the first
day. Nothing a route or a loop step receives changes, and every service is still built by the code that builds it today.

The foundation also lands every guard the later phases lean on, rewrites the rule into its transitional form, and lands
an exemplar area wiring module beside the root call that hands it its adapters, so the first agent to add a service
afterwards imitates the new path. From this landing on, a service new to blizzard is born in the container: the builders
take no new construction. The four members nothing reads are deleted here rather than carried through.

### Adoption: one service at a time, as it is

A service adopts dishka when its construction moves out of the legacy builder and into its area's wiring module,
unchanged in every other respect. Its constructor keeps its signature, and every service that took the bag still takes
the bag. The bag's field for the adopted service is filled by asking the container, so anything still reading the bag
reads the very instance the container holds, and the strangler's two roads never hand out two copies of one thing.

Adoption runs from the leaves up. A service adopts once every collaborator it takes is already in the container, so the
stores and the process's core go first, then the services that take only stores, then the services above them, and the
builder never needs the container to hand back something the builder has not built yet. The hub adopts first; within
each process the natural batch is a concept package's services, leaves first, which keeps each landing to one area of
the code. The phase ends when the builders construct nothing and only assemble bags from what the container hands them.

### Narrowing: from the locator to the seam

Once construction belongs to the container, the consumers of the bags are taught to ask for what they use. A route that
read three services off `TenantServices` names those three services; a service that took a bag to reach into it takes
its collaborators as constructor seams, the shape `bzh:narrow-seams` already demands of the runner. The hub's API edge,
exempt from that rule today, falls under it. The work is again one consumer at a time and needs nothing new from the
container. Splitting `TenantServices` into per-concept sub-bags is not part of it: a consumer that names exactly what it
uses has no need for a smaller bag.

### Cutover: dropping the old path

Cutover is declared rather than discovered: when adoption has emptied the builders and narrowing is near its end, the
owner calls it, and from then the old path is taken apart as fast as nothing rests on each piece of it. The hand-written
builders go first, since by then a bag is one more thing the container can build for itself. Each bag follows the day
its last consumer narrows, along with the bridges that served it. The gates that admitted both paths during the
transition harden to admit only the container — no bag type in a signature, and no process collaborator constructed
outside a provider — and the rule takes its final form in the same change. The epic is delivered when the hub, the
runner host, and the CLI are wired only through their containers.

## What keeps it honest

Each guard lands with the foundation, as a standing gate in the unit tier, because each answers a way dishka can fail
quietly.

- **The graph is checked whole.** A test builds each process's container, and each CLI command group's, with strict
  validation and serves no request, so a service missing a collaborator, or a type with two owners, fails the gate
  rather than a request.
- **Every route resolves.** dishka does not check what only a route asks for, so a test walks every route and resolves
  each service it names at the scope that route runs in.
- **A role is a type.** Where two collaborators share a type, as the virtual clock and the wall clock do, each gets a
  type of its own, so strict validation has a second owner to refuse and the wrong one is never handed out by default.
- **One door into a child container, always locked.** A single helper opens every tenant and runner child container with
  its lock, and the gate refuses any other way in.
- **One instance on every road.** Every port an adapter serves resolves to the same object, and while a bag and the
  container both serve a service, the two hand out the same object too.
- **dishka stays at the edge.** Only the translation modules and the route edge import dishka. A domain package imports
  only the registry interface from `foundation/`, and no domain class is decorated or marked for the container.
- **Wiring modules stay apart.** A wiring module imports its own package's types, its ports, and the registry interface,
  never another area's wiring, and a `register` parameter is an outward port, never another area's service.
- **Teardown is proven in order.** A shutdown test shows each tenant's connections closing before the engine they share
  is disposed.

Two smaller disciplines travel with them. A constructor with default arguments gets an explicit factory rather than
autowiring, since dishka ignores defaults. And blizzard pins dishka's 1.x line and uses only its forms that are not
already deprecated, so the move to 2.0 is one landing of its own when the major ships, not a migration threaded through
every adoption.

## The rule

`bzh:dependency-injection` and `bzh:narrow-seams` in `blizzard-context` change twice. At the foundation they describe
the transition: the container is each process's composition graph, new services are born in it, and the builders and
bags are a path being retired that admits nothing new. At cutover they describe the end state, with the reasoning this
plan carries, so the next person to ask why blizzard has a container finds the answer beside the rule. `bzh:domain-core`
admits the registry interface and names dishka among the frameworks a domain module may not import. The layering gates
in `tests/test_layering.py` that assume hand wiring — the roster of composition roots, and the check that no class is
constructed twice across the hub's root files — are restated against the translation modules and the wiring modules in
the foundation. The exemplars move with the code they model (`bzh:exemplar-conformance`): the exemplar wiring module at
the foundation, and each exemplar service the day its production counterpart adopts.

`epic:strict-types` is in flight beside this one, and dishka's `provide` is partially unknown to pyright in strict mode.
Because only the translation modules import dishka, the two epics agree on one named waiver confined to them, and every
wiring module stays strict-clean through the registry interface.

## Compatibility

Wiring is not a released surface. Nothing an operator, a runner on a supported release, or a client of the API can see
changes: the HTTP routes, the CLI, the configuration, and the wire models are untouched, and every landing preserves
behavior. The removal of the old path is therefore the epic's own last phase, not a follow-up break.

## Open questions

- **The batch size for adoption.** One landing per concept package is the starting assumption; a package whose services
  form a deep chain may want splitting. It is decided when adoption begins.

## Out of scope

The web board, the mock fleet in `blizzard-mock`, and any change to what blizzard does. Splitting the bags into smaller
bags is superseded by narrowing and is not done.
