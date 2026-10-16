# Release engineering

Landing, releasing, and deploying pulled apart: a first real release the dogfood instance runs, compatibility held
against what releases shipped, notes written with the change, a weekly train with a way to patch, `master` kept
releasable behind flags, and artifacts a stranger can trust.

| Where                                      | Read when                                                                                                 |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md)           | Reasoning about the initiative as a whole — the stance, the rhythm, compatibility, and the cut into epics |
| [first-release/](./first-release/index.md) | Cutting the first real release and moving the dogfood instance from `master` onto pinned releases         |
| [compat-floor/](./compat-floor/index.md)   | Holding every compatibility check against the supported releases, and refusing a runner out of its window |
| [notes/](./notes/index.md)                 | Writing change and upgrade notes beside the code, and assembling a release's notes from them              |
| [release-lines/](./release-lines/index.md) | Running the weekly train, patching a released line, cutting a major, and folding the store's history      |
| [dark-launch/](./dark-launch/index.md)     | Landing unfinished work behind a flag, turning it on in a release, and retiring it                        |
| [supply-chain/](./supply-chain/index.md)   | Signing and attesting every artifact, and publishing a runner installable at a named version              |
