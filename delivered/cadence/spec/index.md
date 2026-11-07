# Cadence specifications

The technical contracts behind `epic:cadence`. Product intent begins at the [epic plan](../index.md): that a cadence is
a bucket of chores kept in a project, that every chore is a routine run over its scopes in a mode with a charge note,
that each chore over each scope is an errand judged on its own, that chores and scopes are linked by name and a chore
that can no longer run needs attention rather than being invalid, that a run knows the errand that minted it, that a
schedule and a spacing are the two checks and a spacing counts from a chosen anchor, that an errand has one live run and
no catch-up, that the hub looks once a minute and mints in the cadence's own project, that a chore promotes what it
mints unless it says otherwise, and that an edited chore keeps its errands. This spec states how each of those is built
and redefines none of it, and gives a routine the slug a chore links by.

It builds on what the garden already does. A run started by hand hands a routine and a scope to the garden's run
service, which refuses in a fixed order — a retired routine, a graph that no longer resolves, a scope the routine does
not sweep, a retired scope — settles a `delta` with no baseline down to `full`, and composes the charge with the note as
its *This run* section. The run's work item, its chunk, and its run identity are then written in one transaction, with
no promote fact, so the chunk rests `not_ready`. The hub's only timer is the sweep: a named periodic pass, run per
tenant, and steppable on request. Routines and scopes are configured records (`bzh:configured-record`) declared in the
tenant's config document (`bzh:config-apply`), and a cadence becomes the next record beside them.

Nothing here crosses the fleet wire. A cadence mints at the hub through the path a hand-started run takes, and a runner
claims what it mints as it claims anything else, so neither the hub-to-runner wire nor `blizzard-mock` changes
(`bzh:wire-change-extends-mock` is not engaged). The operator routes it adds regenerate the board's client
(`bzh:generated-client`) and the CLI contract.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, a new domain spoke takes the cadence, the chore, the
errand, when an errand is due, and `bzh:errand-one-live-run` ([due.md](./due.md)); the domain's statement of the promote
gate, in the chunk statuses and in the routine's run, names a promoting chore as the one admission beside an operator's
([mint.md](./mint.md) §Promoting); and the hub's crash-correctness record gains the cadence mint beside the routine-run
mint. Blizzard's operator documents gain a page for cadences, and the pages on config documents, the hub's sweeps,
routines and scopes, and the board each gain what touches them. The generated contract documents — the store's, the
CLI's, the hub configuration's, and the hub's OpenAPI — follow the code.

| Where                                | Read when                                                                                                                                                                                |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [routine-slug.md](./routine-slug.md) | Giving a routine a slug beside its name — the two fields, the transition that fills both from today's name, and the released surfaces that keep answering                                |
| [record.md](./record.md)             | Shaping a cadence and its chores — every field, its validation, its tables, its verbs, and its section of the config document                                                            |
| [due.md](./due.md)                   | Deciding whether an errand is due — its last run, the three anchors, the schedule's moments in a zone, the spacing, and the never-run errand                                             |
| [mint.md](./mint.md)                 | Minting an errand — the cadence sweep, the errand's mark on its run, the author, promoting, the locked write, bounds, tenancy, and the crash                                             |
| [surfaces.md](./surfaces.md)         | Showing a cadence — the attention it needs, its errands and their last runs, the CLI, and the board                                                                                      |
| [carry-over.md](./carry-over.md)     | Landing the epic — the order of its pieces, proving its timing on the controlled clock, the flag that holds the minting dark, the three epics that move over, and what is left for later |
