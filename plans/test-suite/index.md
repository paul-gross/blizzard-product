# Test suite

The service tier moved onto a shared hub and into Playwright Test, one epic per stage of the move, and the gate's speed
beside it.

| Where                                | Read when                                                                                                                   |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md)     | Reasoning about the initiative as a whole — why Playwright, the rules every ported test keeps, and the order the epics land |
| [foundation.md](./foundation.md)     | Building the Playwright project, the shared-hub harness and its fixtures, or reading what the spike proved                  |
| [hub-tier.md](./hub-tier.md)         | Porting a hub-side service test, re-tiering a test that starts no daemon, or replacing a fixed SSE wait                     |
| [runner-tier.md](./runner-tier.md)   | Porting a runner-side service test, or building the runner surfaces those tests need                                        |
| [optimization.md](./optimization.md) | Speeding up the pipeline, the pull request gate, e2e, or the tiers that stay in Python                                      |
