# Release notes specifications

The technical contracts behind `epic:release-engineering/notes`. Product intent begins at the [epic plan](../index.md);
implementation enters at the concern below. What counts as breaking is owned by `blizzard`'s `docs/versioning.md`, and
whether a break may pass by `epic:release-engineering/compat-floor`; when a cut happens and on which line by
`epic:release-engineering/release-lines`; and whether a flag is on by `epic:release-engineering/dark-launch`. This spec
gives each of them a note and redefines none.

Paths are relative to the `blizzard` repository root unless they name another repository.

The epic does not stop at `blizzard`'s source. `blizzard-context` gains the rule for when a note is owed and what it
says, the procedure for writing one, and the gate's check in the verifiability matrix, and `bzh:release` stops asking
for hand-written upgrade notes after the tag ([harness.md](./harness.md)). `blizzard-mock` gains nothing: its changes
reach no operator.

| Where                            | Read when                                                                                                                        |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| [fragments.md](./fragments.md)   | Writing or validating a change note — where it lives, its id, its fields, its body, and what may change once released            |
| [owed.md](./owed.md)             | Deciding whether a change owes a note — the harness's judgement, the gate's mechanical triggers, the waiver, and the `!` pairing |
| [upgrade.md](./upgrade.md)       | Writing an upgrade note, a deprecation, or a removal — the boundary sections of `docs/upgrade.md`, and a major's gathered guide  |
| [assembly.md](./assembly.md)     | Cutting a release's notes — the assembler, the move into a version's directory, the rendered order, and the commit-log appendix  |
| [harness.md](./harness.md)       | Writing the rules, procedure, and matrix row into `blizzard-context`, and changing `bzh:release`                                 |
| [carry-over.md](./carry-over.md) | Landing the epic — the order of landings, seeding the first assembled release, and retiring the placeholder                      |
