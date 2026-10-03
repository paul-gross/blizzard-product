# Runner-spans specifications

The technical contracts behind the runner-spans slice of `epic:tracing`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. The step identity these spans join is owned by
the fleet-spans [span contract](../../fleet-spans/spec/spans.md), and the sweep discipline they share by its
[emission contract](../../fleet-spans/spec/emission.md).

| Where                        | Read when                                                                                                  |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------- |
| [spans.md](./spans.md)       | Shaping the runner's spans: what each is built from, how they parent into the hub's step, their attributes |
| [emission.md](./emission.md) | Building the runner's sweep: where it runs, when a lease is ready to tell, configuration, and verification |
