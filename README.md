# blizzard-product

**The live product intent behind [blizzard](https://github.com/paul-gross/blizzard).** What we are building next, and
why, kept current.

Blizzard is an orchestration platform for autonomous fleets of coding agents, and it is built by its own fleet. Work is
ingested, agents build it, and the result lands. That arrangement moves the interesting decision upstream: a fleet is
only as good as the intent handed to it, so what is worth building is settled here, in writing, before anything is
filed.

So the split is clean. The `blizzard` repo holds the software and the record of how it behaves. This repo holds the
argument for what deserves to exist, in the order it deserves to be built.

Mechanically, this is a [winter](https://github.com/paul-gross/winter) extension. It installs into blizzard's own winter
workspace as `blizzard-product`, so every agent working in that workspace can reach product intent by name and read it
when the task is deciding what to build, rather than carrying it on every task.

## What is here

Intent is promoted in deliberate steps rather than drifting from an idea into a branch. A destination is declared as
something users will be able to do, and it demands the capability areas that serve it. A capability area waits as a
registry row until the work comes within striking distance, at which point it earns a written plan with its scope
decided and its open questions resolved. Only then does anything reach GitHub as a startable issue. Landed work leaves
the registry and enters the ledger.

[index.md](./index.md) is the map: the charter that judges whether a capability is ours to build, the epic and milestone
registries, the plans behind promoted work, the delivered ledger, and the market research we keep against the field.
[MAINTAINERS.md](./MAINTAINERS.md) owns the promotion workflow itself and the rules each registry is held to.

Note what is deliberately absent. Nothing here tracks status. A registry row says a capability is intended and where it
sits in priority, never whether someone is working on it; where work actually stands lives in GitHub and in the
milestone charts. The registries are human-owned: the fleet may argue for a direction, and does, but what enters a
registry and where it ranks is decided rather than accumulated.

## Contributing ideas

Ideas, plans, and thoughts are welcome here, and this is the right door for them. Blizzard itself does not take outside
pull requests, so a proposal that lands in this repo travels further than one aimed at the code: what is accepted here
becomes the intent the fleet builds from.

Open an issue describing the capability and who needs it, in terms of the person with the problem rather than the
implementation you have in mind. [CONTRIBUTING.md](./CONTRIBUTING.md) covers what a proposal is read for, and
[MAINTAINERS.md](./MAINTAINERS.md) covers how accepted intent is promoted from there.
