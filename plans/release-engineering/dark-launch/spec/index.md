# Dark-launch specifications

The technical contracts behind `epic:release-engineering/dark-launch`. Product intent begins at the
[epic plan](../index.md); implementation enters at the concern below. The supported releases, the skew window's
enforcement, and every compatibility check are owned by `epic:release-engineering/compat-floor`; the change note, its
format, and the release's assembled notes by `epic:release-engineering/notes`; the release cut and its branches by
`epic:release-engineering/release-lines`. This spec adds flags to them; it redefines none.

Paths are relative to `blizzard/src/blizzard/` unless they name another root.

The epic reaches beyond `blizzard`'s source. `blizzard-context` gains the procedure under `workflows/`, routed from its
`workflows/index.md`, the rule that a flag hides behavior and never contract under `standards/` beside
`standards/wire.md`, and the lifecycle check's row in `verification/blizzard.md`. `blizzard-mock` serves every route a
flag guards as it serves any other, with no flag of its own (`bzh:wire-change-extends-mock`). Blizzard's operator
documents gain `docs/flags.md`, beside `docs/versioning.md`, describing what an override is and is not.

| Where                            | Read when                                                                                                                   |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| [registry.md](./registry.md)     | Declaring a flag — the registry file, its fields and scopes, naming, tombstones, and the one call code reads a flag through |
| [reading.md](./reading.md)       | Resolving a flag at runtime — the hub, a runner, and the board, overrides, the fleet scope across skew, and visibility      |
| [contract.md](./contract.md)     | Shipping surface behind a flag — what each kind of surface does while off, and why the compatibility checks never read one  |
| [lifecycle.md](./lifecycle.md)   | Holding a flag's life — the lifecycle check, turning on, retiring, testing both sides, and the procedure                    |
| [carry-over.md](./carry-over.md) | Landing the epic — the order its pieces land in, and what of today's fleet it leaves alone                                  |
