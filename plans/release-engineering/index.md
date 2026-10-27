# Release engineering

The weekly train and verifiable artifacts remain: the first release, compatibility against shipped releases, change
notes, and dark-launch machinery are delivered. The dogfood instance uses pinned builds of `master`, moved by hand;
operators' release upgrades remain separate from that policy.

| Where                                      | Read when                                                                                                      |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md)           | Reasoning about the initiative as a whole — the stance, the rhythm, compatibility, and the cut into epics      |
| [first-release/](./first-release/index.md) | Tracing the delivered release machinery and the superseded dogfood-on-releases requirement                     |
| [compat-floor/](./compat-floor/index.md)   | Tracing `epic:release-engineering/compat-floor` — compatibility held against the supported releases, delivered |
| [notes/](./notes/index.md)                 | Tracing the delivered change-note and release-note assembly epic                                               |
| [release-lines/](./release-lines/index.md) | Running the weekly train, patching a released line, cutting a major, and folding the store's history           |
| [dark-launch/](./dark-launch/index.md)     | Tracing the delivered flag machinery and its deferred first-real-flag lifecycle proof                          |
| [supply-chain/](./supply-chain/index.md)   | Signing and attesting every artifact, and publishing a runner installable at a named version                   |
