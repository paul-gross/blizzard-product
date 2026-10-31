# Release engineering

Every epic is delivered: the first release, compatibility against shipped releases, change notes, the weekly train and
patched lines, dark-launch machinery, and verifiable artifacts. The dogfood instance uses pinned builds of `master`,
moved by hand; operators' release upgrades remain separate from that policy.

| Where                                      | Read when                                                                                                      |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md)           | Reasoning about the initiative as a whole — the stance, the rhythm, compatibility, and the cut into epics      |
| [first-release/](./first-release/index.md) | Tracing the delivered release machinery and the superseded dogfood-on-releases requirement                     |
| [compat-floor/](./compat-floor/index.md)   | Tracing `epic:release-engineering/compat-floor` — compatibility held against the supported releases, delivered |
| [notes/](./notes/index.md)                 | Tracing the delivered change-note and release-note assembly epic                                               |
| [release-lines/](./release-lines/index.md) | Tracing the delivered weekly train, patched lines, majors, and store-history fold                              |
| [dark-launch/](./dark-launch/index.md)     | Tracing the delivered flag machinery and its deferred first-real-flag lifecycle proof                          |
| [supply-chain/](./supply-chain/index.md)   | Tracing the delivered artifact signing and attestation, and the runner installable at a named version          |
