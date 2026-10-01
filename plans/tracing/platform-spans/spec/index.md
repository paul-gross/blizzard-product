# Platform-spans specifications

The technical contracts behind the platform-spans slice of `epic:tracing`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. The step identity these spans nest under is owned
by the fleet-spans [span contract](../../fleet-spans/spec/spans.md).

| Where                                      | Read when                                                                                                                |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| [nesting.md](./nesting.md)                 | Carrying a step's trace identity into a worker, through its commands, and on to the runner and hub                       |
| [instrumentation.md](./instrumentation.md) | Instrumenting a daemon, the CLI, or a store — what is traced, what is sampled, what is configured, and what never leaves |
