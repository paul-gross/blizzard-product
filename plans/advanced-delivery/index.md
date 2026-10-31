# Advanced delivery

Finished work that lands the way its project needs it to land: a forge named once and spoken through one seam, a flow
per branch, a landing policy per repository, GitLab beside GitHub, and merge trains. The hub's own actions, the forge
seam, and per-branch flows are delivered, and the remaining epics build on them.

| Where                            | Read when                                                                                                  |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md) | Reasoning about the initiative as a whole — the shape of delivery it moves to, what it builds, and its cut |
| [actions/](./actions/index.md)   | Tracing the delivered hub-action execution boundary and retirement of `run:`                               |
| [forges/](./forges/index.md)     | Tracing the delivered per-tenant forge declaration and the landing seam every forge sits behind            |
| [flows/](./flows/index.md)       | Tracing the delivered resource-scoped steps and the per-branch flow chunks submit to                       |
| [policies/](./policies/index.md) | Giving each repository a landing policy and each project a landing order, on GitHub                        |
| [gitlab/](./gitlab/index.md)     | Putting GitLab behind the seam and landing across kinds of forge                                           |
| [trains/](./trains/index.md)     | Batching ready work into a train proven once, or handing the line to the forge's own queue                 |
