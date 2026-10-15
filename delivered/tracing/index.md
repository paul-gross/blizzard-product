# Tracing

The fleet's own account of itself, sent as OpenTelemetry spans to whatever backend the operator names — one epic per
layer of a node step the spans describe. Each epic is frozen as it shipped.

| Where                                        | Read when                                                                                                 |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md)             | Reasoning about tracing as a whole — why the harness cannot answer it, the span shape, and its boundaries |
| [fleet-spans/](./fleet-spans/index.md)       | Looking up the hub-told trace per node step and the step identity every other span joins                  |
| [platform-spans/](./platform-spans/index.md) | Looking up the spans for an agent's calls to the platform, nested inside the step that made them          |
| [runner-spans/](./runner-spans/index.md)     | Looking up the runner-told inside of a step — each harness invocation, wait, and check                    |
