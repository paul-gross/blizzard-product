# Fact egress

The fleet's facts leaving the hub as files an operator's own tools read, one epic per dataset. Each epic is frozen as it
shipped.

| Where                            | Read when                                                                                                        |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md) | Reasoning about fact egress as a whole — what blizzard owes and does not, the grain, and what must never be true |
| [steps/](./steps/index.md)       | Looking up the exporter and its step and invocation datasets, built on the step definition tracing uses          |
| [events/](./events/index.md)     | Looking up the dataset of transcript-derived events and the ledger that keeps it honest                          |
