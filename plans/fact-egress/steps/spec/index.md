# Steps specifications

The technical contracts behind the steps slice of `epic:fact-egress`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. What a step is, and when it closed, is owned by
the fleet-spans [span contract](../../../tracing/fleet-spans/spec/spans.md).

| Where                    | Read when                                                                                              |
| ------------------------ | ------------------------------------------------------------------------------------------------------ |
| [rows.md](./rows.md)     | Shaping a row — the `steps` and `invocations` datasets, their columns, and what never leaves           |
| [export.md](./export.md) | Building the exporter — the sweep, cursors, the directory and its files, formats, configuration, proof |
