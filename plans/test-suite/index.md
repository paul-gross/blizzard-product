# Test suite

Every tier above the component tier moved into Playwright Test and onto a few long-lived shared hubs, one epic per stage
of the move, and the gate's speed beside it.

| Where                                    | Read when                                                                                                                               |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md)         | Reasoning about the initiative as a whole — why Playwright, the rules every ported test keeps, and the order the epics land             |
| [foundation/](./foundation/index.md)     | Building the Playwright package, the shared-hub harness, the mock fleet and rulebook changes it needs, or reading what the spike proved |
| [hub-tier/](./hub-tier/index.md)         | Tracing the delivered hub-side ports, which service tests stay in Python, or how a fixed SSE wait was replaced                          |
| [runner-tier/](./runner-tier/index.md)   | Porting a runner-side service test, or building the stepping host and other runner surfaces those tests need                            |
| [fleet-tier.md](./fleet-tier.md)         | Moving an e2e scenario into the fleet service tests or a browser scenario into the UI tests, or retiring the e2e tier                   |
| [optimization/](./optimization/index.md) | Speeding up the pipeline, the pull request gate, or the tiers that stay in Python                                                       |
| [journey.md](./journey.md)               | Giving the daemons a consistency check of their own, porting the journey to Playwright, or guarding `blizzard-mock`'s wire parity in CI |
