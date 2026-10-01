# Fleet-spans specifications

The technical contracts behind the fleet-spans slice of `epic:tracing`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below.

| Where                        | Read when                                                                                                    |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [spans.md](./spans.md)       | Shaping a trace — its identity, the spans inside it, the facts each is built from, and the attribute set     |
| [emission.md](./emission.md) | Building the sweep that sends them — cursor, settling, export, configuration, replay, and how it is verified |
