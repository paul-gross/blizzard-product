# Plan — `epic:self-sourced-work`

The forge stops being the only door work can arrive through. This epic builds the door itself: work items that live in
the hub's own store, addressed like any other work (`hub:42`), flowing through the same ingest → backlog → promote →
deliver pipeline as forge-born chunks. The producers that will one day author work through it — a delivery-review pass
that judges whether a landing truly succeeded, a retrospective whose finding becomes an item rather than a paragraph —
arrive through later epics; what this epic guarantees is that when they do, stating work is a structured proposal on a
node completion, and the hub materializes it at delivery. And the door pays for itself before any producer shows up: a
hub with zero forge sources configured becomes fully operable — items authored by hand, a queue with no forge behind it
at all.

## What already holds

The work-source protocol fits an internal source without alteration. `IWorkSource` asks for `parse`, `fetch`, `label`,
`web_url`, and `branch_url`; a chunk stores only the opaque `{source, ref}` pointer and every consumer fetches content
fresh from the source — so a source whose `fetch` reads the hub's own table satisfies the whole contract, and nothing
above the seam can tell the difference. The two address methods are the seam's own escape hatch at work: each may answer
with nothing when a pointer has no such address, which is the honest answer for a source with no repository and no forge
to link out to. Write abilities are already sibling protocols a source may or may not implement (`IWorkAnnotator`,
`IWorkCloser`), which is exactly the shape item mutation needs.

What does not fit is the assembly below the protocol. The registry is built by walking the configured source entries and
resolving each one's declared provider to an adapter, with credentials drawn from the environment; a source that is
always present, holds no credential, and appears in no configuration has nowhere in that walk to come from. Its write
halves are gated the same way, by a per-entry opt-in flag rather than by anything the source itself declares. The seam
is right and the construction path is missing — that gap is scope this epic carries, named below.

Intake is push and deliberate: ingest mints a resting `not_ready` chunk, and promotion is a separate human act. That
gate does not move. The ranking machinery is nearly general — positions are keyed by chunk, and a chunk with no explicit
position falls back to when it was promoted, or to when it was minted if it never was. A backlog chunk has only ever
known the last of those, so ranking one is a question the existing sort key can already answer. What is not general is
the reach: the reorder service reads the ready set directly, so the ready-only restriction is a property of the domain
rather than a guard at the API edge. Opening ranking to the backlog is a change to that service, not a validation lifted
off an endpoint.

Identity comes in exactly the two families authorship needs: hub-local users with roles behind the operator surface, and
runner principals valid only on the fleet router — so who authored an item is something the hub stamps from what it
already knows, never something a caller claims.

And chunks already know how to disappear without deleting anything: a grouped chunk records a fact and becomes ephemeral
— gone from every view, row intact.

## The item and its author

A hub work item carries a per-source monotonic integer `ref` (the `42` in `hub:42`, never reused), a `title`, a markdown
`body` that is the work's full spec, an `author`, an advisory `stated_priority` (`low`/`normal`/`high`), creation and
last-edit instants, and a closure — `delivered` when its chunk lands, `withdrawn` when it is deleted. The author is
structured, not a string: kind `user` with the hub user's login, or kind `fleet` with the runner, chunk, and node the
hub resolves itself when it materializes a proposal at delivery. Comments stay out of the table: the fetch seam returns
an empty list and every consumer is satisfied; an append-only comments table slots in behind the same seam the day a
real need appears.

## What to build

- **The `hub` source, reserved and built-in.** Always present under the reserved name `hub` — no config stanza and no
  credentials, which means giving the registry's construction a way to seat a source that configuration never mentions
  and to hand it a closer that no opt-in flag switches on. Configuration learns to refuse `hub` as a source name, the
  way it already refuses a duplicate one, so nothing a user writes can shadow the built-in. Its `label` prints `hub:42`,
  its `web_url` points at the board's own chunk view rather than anywhere external, and its `fetch` reads the item table
  — which means editing an open item shows up on its chunk immediately, the same fresh-fetch semantics forge issues have
  today.
- **Source-addressed item routes.** Items are a sub-resource of the source they live in: `GET`/`POST` on
  `/api/work-sources/{source}/items` and `GET`/`PATCH`/`DELETE` on `/api/work-sources/{source}/items/{ref}`, plus a
  sources listing so the board can enumerate what exists and what is writable. Tokens (`hub:42`, `blizzard#123`) are
  input grammar, not identity: the CLI parses them once at the edge through the source's own `parse` and the wire
  carries `{source}` and `{ref}` as plain segments. Mutation is capability-gated the way annotate and close already are
  — a sibling editor protocol the hub source implements and the GitHub source does not, so `PATCH` against a forge item
  refuses cleanly today while leaving the seam open for a source that earns it later.
- **Proposals, materialized at delivery.** The fleet never creates items — runners state facts and desires, the hub
  materializes consequences, and this epic keeps that invariant intact. A node attaches proposed work items to its node
  completion as structured payloads — `create` with title, body, and stated priority, or `update` appending evidence to
  an existing open item — riding the same atomic channel assets already ride, so the proposal set inherits the
  completion lane's at-least-once idempotence and a retried completion can never double-propose. When the chunk
  delivers, the hub mints the accumulated proposals into real items, stamping authorship and lineage itself,
  epoch-fenced like the rest of delivery. A graph that wants human judgment places its existing gate construct before
  delivery and the gate presents the proposal docket — strike some, pass the rest — so filtering happens before durable
  state exists, not by deleting items after. Which nodes may carry proposals at all is graph policy. A chunk that never
  delivers takes its proposals with it: a stopped or failed run's opinions do not outlive it — a decision, not an
  accident. This lane is the seam the producer epics use; proposing work is never the part a producer has to build.
- **Auto-ingest, promotion untouched.** Creating an item mints its `not_ready` chunk in the same act — the item is on
  the board the moment it exists, and nothing about it runs until an operator promotes it. Stated priority renders at
  triage as the author's claim; it never writes a queue position.
- **A sortable backlog.** Teach the reorder service to work over the backlog as well as the ready set, so drag-and-drop
  ranks `not_ready` chunks exactly as it ranks the queue. The position machinery already treats rank as a property any
  chunk can carry, and a backlog chunk's mint instant already sorts it sensibly among its peers; what changes is which
  chunks the service will consider neighbours, and the endpoints follow. Ranking the two lists together is not asked for
  here — each is ordered within itself — which keeps a backlog chunk's mint instant and a queued chunk's promotion
  instant from being compared as though they measured the same thing. This makes the backlog a real triage surface
  instead of a list ordered by accident of mint time.
- **Deletion, and why it is not the verb we already have.** An operator can already abandon a chunk by stopping it and
  close one by hand by marking it done, so a third way to make work disappear owes an account of itself. The account is
  what each verb says about the work: stopping says this run was abandoned, completing says the work is finished, and
  both leave the item behind the chunk standing as though it still wants doing. Deletion is the only one that says the
  work should never have been filed — and once items live in the hub, that sentence has to reach the item too, not just
  the card in front of it. A `chunk.deleted` fact makes a chunk ephemeral, following the precedent both grouping and
  manual completion set: nothing is row-deleted, the activity feed can say who deleted what, and a mistake is
  diagnosable. The guard is the unacquired predicate grouping already trusts — backlog and queue chunks delete, anything
  a runner holds refuses by name. On the board: a delete affordance with confirmation on unacquired chunk cards, gated
  like the other chunk controls, announced over the existing change stream. A hub-born chunk and its items live and die
  together — deleting either withdraws the other — while deleting a forge-born chunk merely un-tracks it and the issue
  lives on at the forge.
- **Refine by CLI, view read-only.** `blizzard hub item create`/`edit`/`delete` verbs, thin over the routes, accepting
  the token forms a human types. The board renders the item in full — markdown body, authorship line, stated priority —
  and grows no editor by design: the board is the read-only view, and editing is the agentic surface's job.
- **Closure on delivery.** The `hub` source implements the closer protocol against its own table, so the reconciler that
  closes forge issues marks hub items `delivered` when their chunk lands — items get a complete lifecycle from the
  machinery that already runs.

## What this epic is not

Not the producers: no node proposes work yet — the delivery-review pass, the tending passes, and the retrospective's
finding-to-item rewiring are their own epics, and this one's proposal lane is the seam they will use. Not
auto-promotion: a stated priority is advice for the triaging human, and the promote gate holds against fleet-authored
work exactly as it holds against forge-born work. Not fleet mutation routes of any kind: item creation and editing
through the API stay operator acts, and withdrawal stays a purely human one. Not comments, until something needs them.
Not forge editing: the capability gate leaves that door framed but unopened.

## Open questions

- Whether promotion should read the stated priority to auto-place a high-priority chunk near the front instead of at the
  tail — deferred until real triage shows whether the drag is a burden worth saving.
