---
epic: transactions
refinement: scaffolded
status: horizon
---

# Plan — `epic:transactions`

An agent adding a write to the hub (`persona:fleet-agent`) has to learn which of several ways to change the store this
write should take, and the code offers it every one of them as a precedent. It can open a unit of work and declare its
locks, which is the path the harness asks for. It can open a locked handle through `IChunkExclusiveWrites` and call one
of the `*_locked` twins beside the method it actually wanted, which is the path most of the existing decisions still
take. It can call a repository method that opens its own short write. It can find a store method that decides the
business rule itself, checking that a chunk is terminal inside its own `WHERE`, and copy that. And it can find an event
announced from deep inside a store write, where every other event is announced by the caller once the write has
committed. Each path is correct where it stands. Together they teach a newcomer that any of them will do, and the rules
that say otherwise are written as a list of exceptions a reviewer has to remember.

The debt has a shape that can be counted. Measured on `master` at `7d984dad8`, 25 repository port methods exist only as
a `*_locked` twin of a write, and 17 domain and delivery modules reach the store through the legacy locked seam, where
12 open the unit of work directly and several do both. Four store methods still judge a domain predicate rather than
refusing only a stale write. Two exclusions live in the hub process's memory rather than in the store: the per-tenant
lock that keeps the dependency graph free of cycles, and the lock that keeps an operator's reset of the export cursor
from being overwritten by a pass in flight. Everything holds today, because one hub process serves the store and every
test runs on SQLite, whose single writer lock serializes everything whether or not the code asked it to.

This epic brings blizzard to the final state of how it changes its own data. A write opens one way. A decision that must
win exactly once locks what it reads, in one declared order, through the one unit of work. The store refuses a write
only because it is stale, and the domain alone decides whether it is allowed. An event is announced only after the write
it describes has committed, by the code that asked for the write. Every exclusion a writer depends on lives in the
store, where a second process would meet it, rather than in a process's memory, where it would not. And the locking that
SQLite makes true by construction is shown to hold on Postgres.

## The end state

**One way to open a write.** `IHubTransactions.begin` opens every multi-repository change, and its single lock
declaration — chunks, then keys, sorted — is the only way a unit takes a lock. Every repository write takes
`transaction: Transaction | None`, joining the caller's unit when it is given one and opening its own short write when
it is not, and a write correct only under a lock takes a `LockedTransaction` with no default. The legacy seam goes:
`IChunkExclusiveWrites`, `ILockedChunkRead`, `ILockedWorkRefRead`, `IWriteGraphRepository.locked_name`, and every
`*_locked` twin, each folded into the plain method it shadowed. The guard reads a locked handle offers today — a chunk's
facts and record, its route, its standing edges, a runner's registration — become ordinary repository reads that join
the unit.

**The store refuses stale writes and judges nothing.** The terminal check inside `fence`, the escalation that runs that
fence on its own write, the backlog write that drops a promoted chunk's position, and the promote that writes nothing
for an already-promoted chunk each hand their fact back to the domain, which decides, and the store conditions its write
on the token the domain read. Where a decision is covered by one version or monotonic token, it takes the
compare-and-set form before the locked form, as `bzh:write-reestablishes-read` already prefers.

**One way to announce a change.** A unit returns what it changed, and its caller publishes from that after the commit.
The store's own path to the broker — the publisher a scoped write carries and the callback a store queues on its commit
— is retired, and the event it announces today moves to the caller.

**Adapters stay apart.** An adapter that writes another repository's facts through a sibling's private helper hands that
write back to the domain, which composes both repositories in one unit. `WorkItemStore`'s composite mints are the
largest case.

**Every exclusion lives in the store.** The dependency graph's cycle check, the delete and release that share its lock,
and the export cursor's pass lock each move onto a lock the store holds, so that the in-process locks
`bzh:store-exclusive-write` lists as residual debt are gone. The sweep locks, which order a process's own bookkeeping
and guard no store decision, stay as they are.

**Shown on Postgres.** The focused `blizzard:postgres-uow-concurrency` proof grows to cover what SQLite cannot show: the
closed-tenant check a scoped write opens with, the ordered locks that keep two closes from closing the last two open
tenants, the drain's delete order under enforced foreign keys, and keyed locks of different keys not excluding each
other. A gap the proof closes leaves `gaps.md`, and a mechanism found broken is fixed in this epic. The proof stays a
focused method rather than a Postgres test tier, as the gap ledger already decides.

**The rules take their final form.** `bzh:transaction-participation`, `bzh:store-exclusive-write`, and
`bzh:store-refuses-stale` lose their legacy clauses and their Known debt, and each departure they now catch at review
gains a mechanical gate: a `*_locked` name or a locked-handle type under `hub/`, a store write that reads a
status-family fact to refuse, a broker call inside a store write.

## How it lands

The epic moves one concept area at a time, and every landing preserves behavior.

The store's judgments go first, because each one sits inside a write that the seam convergence will later move, and
moving a write that still decides a rule moves the rule with it. The single path for announcing a change lands beside
them.

The seam converges next, area by area, along the ports that hold the twins: a chunk's lifecycle, its movement, its
route, its dependencies, its decisions and escalations, its hub execution, its record and work refs, and the graph. Each
landing gives the plain method its `transaction` parameter, moves every caller in that area onto
`IHubTransactions.begin`, and deletes the twin. The seam itself is deleted in the landing that moves its last caller.

The exclusions move into the store after the seam has converged, since the cycle check's holders are among the last
callers of the legacy seam. The Postgres proof grows alongside each piece it covers, and the rules take their final form
in the landing that removes the last thing their legacy clauses describe.

## Compatibility

The write path is not a released surface. No route, command, wire model, or configuration changes, so removing the old
path is this epic's own last phase rather than a follow-up break. A lock that needs a store row no table holds today
lands as an expansion (`bzh:store-expand-contract`), and a row in `keyed_locks` needs no revision at all.

## Open questions

- **Whether the tenant row stays a mutex.** Every write in a tenant opens by locking that tenant's row and holds it to
  commit, so on Postgres every writer in one tenant already queues behind the one before it, and the chunk and keyed
  locks beneath never contend. Keeping it is simple and safe, and it makes those finer locks a second line of defence.
  Relaxing it lets writers on different chunks run side by side, but it costs an existence read on every write to keep
  refusing a closed tenant, and it makes every finer lock load-bearing for the first time. The plan assumes the row
  stays a mutex here, and that relaxing it is a decision of its own, taken when a deployment on Postgres needs the
  concurrency.
- **Where the cycle check's lock lives.** One keyed lock per tenant would serialize every dependency change in that
  tenant, as the in-process lock does today, and needs no revision. A lock scoped more narrowly needs a way to name the
  part of the graph a change can close a cycle through. It is decided when the exclusions move.

## Out of scope

Running a second hub process. This epic makes the store the only arbiter of a write, but the tenant cache, the event
broker, the background sweeps, and the boot recovery of open turns stay single-process by construction. The runner's
store, which one process serves by design and whose pool draw is held by the host's own lock. Crash correctness, which
`blizzard-context`'s crash-correctness rules own separately from any lock.
