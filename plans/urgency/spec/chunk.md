# A chunk's urgency

## The column

A chunk carries its urgency by id, in a new `urgency_id` column on the chunk's row. It is an operator-editable property
beside the graph pin and the model, effort, and harness defaults: a plain value overwritten in place, not a fact log,
since no question the plan asks needs a history of what a chunk's urgency used to be.

The column is nullable in the schema, because a revision only expands (`bzh:store-expand-contract`), and never null in
fact: the revision that adds it fills every row ([carry-over.md](./carry-over.md) §Every tenant has `standard`), and
every mint writes it from then on. The chunk's domain model holds an urgency as a required value.

## The chain

Every mint resolves its chunk's urgency down one chain, taking the first link that names one:

1. **The urgency the request states.** Ingesting work (`POST /api/chunks`), authoring a hub item with its chunk,
   accepting a garden proposal, and starting a routine run by hand each take an optional `urgency`, by id or slug.
2. **The chore's override**, for a run a cadence mints.
3. **The routine's default**, for a run started by hand or minted by a cadence.
4. **The tenant's default** ([record.md](./record.md) §The tenant's settings).

The chain is one pure function of the urgency package, handed what each link names and the tenant's live set, and every
mint path calls it: the shared mint preparation the item-backed paths, the routine run, and the cadence mint already go
through gains the urgency beside the routine's model, effort, and harness defaults, and the ingest decision, which mints
without that preparation, gains it too. The chunk's mint takes the resolved urgency as a required argument, so no path
can mint a chunk without one.

A stated urgency that is unknown is refused `422 urgency_unknown`, and a retired one `422 urgency_retired`, before
anything is written. A link lower in the chain never names a retired urgency, because retiring one clears every routine
default and chore override that named it and the default cannot be retired at all; the mint confirms it anyway, under
the lock below.

## Setting an urgency

Every write that sets a chunk, a routine's default, a chore's override, or the tenant's default to an urgency — a mint,
an edit, an apply — takes that urgency's keyed lock among the locks its unit already declares and confirms there that
the urgency is still live (`bzh:write-reestablishes-read`), so it can never land in the moment the urgency is retired
([record.md](./record.md) §Retiring, with the work moved). A mint path that writes without a locked unit today gains one
for this.

## The routine's default and the chore's override

A routine gains `default_urgency`, nullable: an urgency, or none, which takes the tenant's default. It appears on the
routine's create, edit, and views, on its config document entry by slug, and in its export, and is stored as
`default_urgency_id` on the routine's row. A routine run started by hand gains an optional `urgency` in its request,
which outranks the routine's default for that run alone.

A chore gains `urgency`, nullable: an urgency, or none, which takes the routine's default. It appears on the chore's
document within its cadence, on `chore add` and `chore edit`, and on the cadence's views, and is stored as `urgency_id`
on the chore's row. An edit that changes only a chore's urgency keeps the chore's id and its errands' last runs, as any
edit keeping its key does. A chore that promotes still lands its run at the tail of the ready list by rank; its urgency
decides only the group the run competes in.

Both are written under the urgency's lock, both are refused a retired or unknown urgency as a stated one is, and both
read as none once the urgency they named is retired.

## Editing

`PATCH /api/chunks/{chunk_id}` gains `urgency`, by id or slug, under the permission the route already asks,
`chunk:control`. It is editable at every status but `done` and `stopped` — in the backlog, the ready queue, waiting on
or needing a person, paused, running, or delivering — since an open chunk's urgency still decides where it stands should
it return to the queue. A finished chunk refuses the edit as it refuses any edit outside a field's window. An explicit
`null` is refused, and an edit to the urgency the chunk already carries writes nothing.

An edit changes nothing else about the chunk. Its rank stays where it was in its list, so an operator who marks a
backlog chunk urgent and then promotes it sees it land where promotion always lands it, at the tail of the ready list by
rank, and so at the tail of claim order's urgent group. After the commit the chunk publishes its change with cause
`edited` and the queue publishes its change, since claim order may have moved; the edit route publishes no queue change
today, and gains that one.

## What keeps an urgency

Grouping folds chunks into a survivor that keeps its own urgency; the folded chunks' urgencies vanish with them, as
their ranks do. A survivor is never raised to the most urgent of what it absorbed: the fleet does not promote work on
the operator's behalf, and an operator who wants the survivor urgent says so with one edit.

Detach, a requeue, a release, a restart, a pause and its resume, and a migration all leave the urgency as it was. A
chunk returning to the ready queue returns to its own group.

Once a chunk is `done` or `stopped` its urgency is history. A later hand-completion of a stopped chunk keeps it, and no
retire moves it.

## In the chunk's views

Every view of a chunk — its summary on the chunks list, its detail, and its entries on the queue and backlog pages —
gains an optional `urgency` naming the urgency's id and slug. Optional, because a release's client reads views from a
hub that predates the field; from a hub carrying this epic it is always present. The board joins it to the tenant's
urgency set for the name, the color, and the mark, so a renamed or recolored urgency changes every card at once without
a chunk read changing ([surfaces.md](./surfaces.md)).
