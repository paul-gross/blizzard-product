# Compat floor specifications

The technical contracts behind `epic:release-engineering/compat-floor`. Product intent begins at the
[epic plan](../index.md), and the stance it deepens — the contract is what a release shipped, the window drawn around
majors, expand then contract — at the [initiative](../../initiative.md). The window's policy is
`epic:release-engineering/first-release`'s to restate in `docs/versioning.md`; this spec declares it where code can read
it and enforces it, and redefines none of it. The breaking marker's upgrade note is `epic:release-engineering/notes`';
the release branches a patch is checked on, the mock's mirrored tags, and opening `master` for a major are
`epic:release-engineering/release-lines`'. A flag hides behavior, never contract
(`epic:release-engineering/dark-launch`), so nothing here reads a flag.

Paths are relative to the `blizzard` repository's root.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, `architecture/system-shape/fleet-wire.md`'s
`bzh:fleet-wire-additive` is restated against the supported releases rather than the previous minor and the last `edge`
publish, and generalises into `bzh:released-surface-additive` covering every surface below; `standards/persistence.md`
gains `bzh:store-expand-contract` beside `bzh:manual-migrations`; `verification/blizzard.md` replaces
`blizzard:wire-compat` with `blizzard:compat` and adds `blizzard:rehearsal`; and `workflows/` gains the procedure for
judging a break, beside `bzh:release` ([gates.md](./gates.md) §The procedure). In `blizzard`'s own docs,
`docs/versioning.md`'s skew-window section stops describing the per-commit check and points at `blizzard:compat`, and
`docs/ci.md` describes the jobs [gates.md](./gates.md) places. `blizzard-mock`'s `mock_hub` mirrors the handshake's
fields and refusal (`bzh:wire-change-extends-mock`).

| Where                            | Read when                                                                                                                |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| [baseline.md](./baseline.md)     | Computing the supported releases — the window file, the tags, and how a release's surface is read from its tag           |
| [surfaces.md](./surfaces.md)     | Building a surface's contract document and its classifier — wire, HTTP API, configuration, traces, egress, store         |
| [gates.md](./gates.md)           | Placing the check — what runs where, what lets a break through, the old modes retired, and the procedure for judging one |
| [rehearsal.md](./rehearsal.md)   | Building the upgrade rehearsal — which releases, how each is installed and seeded, the journey, the store walk           |
| [handshake.md](./handshake.md)   | Adding the version handshake — what registration and the host report carry, the window rule, the refusal, the board      |
| [carry-over.md](./carry-over.md) | Landing the epic — its order after the first release, the first floor, and a fleet whose runners predate the handshake   |
