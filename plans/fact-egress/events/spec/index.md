# Events specifications

The technical contracts behind the events slice of `epic:fact-egress`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. The exporter this slice adds a dataset to — its
sweep, directory, formats, delivery and operator surface — is owned by the steps slice's
[export contract](../../steps/spec/export.md).

| Where                    | Read when                                                                                               |
| ------------------------ | ------------------------------------------------------------------------------------------------------- |
| [rows.md](./rows.md)     | Shaping the `events` dataset — its record types, columns, the current-truth view, and what never leaves |
| [export.md](./export.md) | Exporting it — the derivation cursor, the drop fact this slice adds, extractor versions, and proof      |
