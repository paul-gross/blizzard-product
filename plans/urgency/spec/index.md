# Urgency specifications

The technical contracts behind `epic:urgency`. Product intent begins at the [epic plan](../index.md): that urgency is a
second axis beside rank, drawn from a short ordered set each tenant keeps; that an urgency holds an identity, a slug
fixed at create and never reused, and a display name free to change; that a tenant setting names the default every new
chunk takes; that every open chunk carries a live urgency and no surface can empty it; that retiring an urgency asks
where its open work goes, finished work keeping what it ran under, and a retired urgency can be revived under its own
slug; that a routine's default and a chore's override fall back a level rather than through; that the configuration
document declares all of it and refuses a retired urgency rather than reviving one; that every tenant has `standard` on
the day this lands; that work can arrive already urgent; that a runner claims the most urgent group first and rank
within it, without borrowing urgency across a prerequisite; and that the board explains each runner's next claim. This
spec states how each of those is built and redefines none of it.

It builds on what the hub already does. A tenant holds one `not_ready` list and one `ready` queue across all its
projects, each ranked within itself by append-only position facts, the newest per chunk deciding and the instant it
entered the list standing in for a chunk never positioned (`bzh:ranking-is-per-list`). The hub decides what a runner
claims: a runner peeks, the hub walks its ready order skipping projects the runner does not serve and chunks whose
prerequisites are unfinished or whose node the runner cannot run, holding at the first unusable chunk or passing over it
as the runner's policy asks, and hands back one entry, which the runner then claims by route. Forges, work sources, and
repositories are tenant-wide configured records (`bzh:configured-record`), and routines and cadences are project-owned
ones, each declared in the tenant's configuration document (`bzh:config-apply`). An urgency is the next tenant-wide
record beside them, and the first whose slug, like a routine's, is fixed at create.

A work item's stated priority — `low`, `normal`, or `high`, carried from its source or set when a hub item is authored —
is the source's word about an item and stays exactly that. Urgency never reads it and never writes it: one is what the
tracker says, the other is what the operator has decided about how soon a runner should reach for the work.

The fleet wire changes only additively. The order the hub walks changes, and that order was always the hub's to decide
([claim-order.md](./claim-order.md) §Runners of every release). The chunk's detail and the queue's entry, which runner
routes share with the board, each gain an optional `urgency` (`bzh:fleet-wire-additive`), so `blizzard-mock`'s mirror
learns the field in the same piece (`bzh:wire-change-extends-mock`). The operator routes the epic adds regenerate the
board's client (`bzh:generated-client`), the CLI contract, and the store contract.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, a new domain spoke under the work hub takes the
urgency, the tenant's default, retiring with its work moved, and claim order; the ranking spoke names claim order as the
order a runner reaches in, rank within urgency, beside the order each list keeps; the acquisition spoke's walk is
restated over claim order; the configuration spoke gains the urgency's and the tenant settings' entries, with their
departures from the shared verb set — an ordered set, a retire that takes a body and can be refused, a keyless singleton
— and from `bzh:config-apply`, which revives a retired record where a document naming a retired urgency is refused; the
domain spokes on routines and scopes, on cadences, and on tenancy gain a routine's default, a chore's override, and a
new tenant's `standard`; and the chunk spoke gains the urgency among a chunk's operator-editable properties. Blizzard's
operator documents gain a page on urgencies, and the pages on the board, the config document, routines and scopes, and
cadences each gain what touches them.

| Where                              | Read when                                                                                                                                                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [record.md](./record.md)           | Shaping an urgency or the tenant's settings — every field and its validation, the tables, the verbs, the set's order, retiring with work moved, and the config document     |
| [chunk.md](./chunk.md)             | Giving a chunk its urgency — the column, the chain every mint path resolves, the routine's default and the chore's and run's override, editing, grouping, and finished work |
| [claim-order.md](./claim-order.md) | Ordering what a runner reaches for — claim order, the peeks that walk it, hold and pass-over, prerequisites, and the next-claim explanation                                 |
| [surfaces.md](./surfaces.md)       | Showing urgency — the views, the administration screen, the pill on cards and list rows, a chunk's urgency control, a runner's next claim, and the CLI                      |
| [carry-over.md](./carry-over.md)   | Landing the epic — the order of its pieces and their proof, the revision that gives every tenant `standard`, versions in the field, and what the epic leaves to tagging     |
