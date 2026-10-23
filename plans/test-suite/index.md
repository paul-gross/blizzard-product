# Test suite

The service tier's move into Playwright Test and onto a few long-lived shared hubs, one epic per stage of the move, and
the gate's speed beside it.

| Where                                    | Read when                                                                                                                               |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md)         | Reasoning about the initiative as a whole — why Playwright, the rules every ported test keeps, and the order the epics land             |
| [foundation/](./foundation/index.md)     | Building the Playwright package, the shared-hub harness, the mock fleet and rulebook changes it needs, or reading what the spike proved |
| [hub-tier/](./hub-tier/index.md)         | Porting a hub-side service test, deciding which service tests stay in Python, or replacing a fixed SSE wait                             |
| [runner-tier/](./runner-tier/index.md)   | Porting a runner-side service test, or building the stepping host and other runner surfaces those tests need                            |
| [optimization/](./optimization/index.md) | Speeding up the pipeline, the pull request gate, e2e, or the tiers that stay in Python                                                  |
