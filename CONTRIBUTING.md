# Contributing

This repo holds blizzard's product intent: what is worth building next, and why. Contributions here are arguments and
proposals rather than code, and they are welcome from anyone.

Start with [README.md](./README.md) for what the repo is, and [index.md](./index.md) for where each kind of document
lives.

## What belongs here

| Bring it here                                                  | Bring it to [blizzard](https://github.com/paul-gross/blizzard) |
| -------------------------------------------------------------- | -------------------------------------------------------------- |
| A capability blizzard does not have and someone needs          | A bug in blizzard as it behaves today                          |
| A direction you think the product should take, or should not   | A question about running or configuring an existing release    |
| A case that a planned capability is scoped wrongly             | A security report                                              |
| Research or prior art on a problem blizzard is trying to solve | Anything that is already a fix rather than an intent           |

Blizzard's code is built by its own fleet and takes no outside pull requests, so a proposal accepted here travels
further than a patch aimed at the code.

## Raising a proposal

Open an issue on this repo. A proposal is read for whether the capability is blizzard's to build, so make that case
rather than a case for an implementation:

- **Lead with the person and the problem.** Who hits this, and what does their day look like without it? A proposal that
  opens with a mechanism is hard to judge, because the reader cannot tell what would count as solving it.
- **Say what changes for them.** Describe the outcome you would be able to point at, not the feature you imagine
  producing it.
- **Name the boundary.** What is deliberately not included, and what would make this the wrong thing to build.
- **Bring evidence where you have it.** A concrete moment it went wrong, a workaround you are maintaining, or prior art
  worth reading is worth more than a strong adjective.

[charter/mission.md](./charter/mission.md) is the standard a proposal is measured against, and
[charter/personas.md](./charter/personas.md) names the people the product is built for. Reading both first will tell you
whether an idea is in scope, and will sharpen it if it is.

## What happens next

An accepted proposal is promoted in steps. It enters [epics.md](./epics.md) as a capability area with a stable id and a
position in priority order. When the work comes close enough to start, it earns a written plan under `plans/` with its
scope decided. Only then is it filed to GitHub as startable issues, which the fleet picks up. Once it has landed, it
leaves the registry and enters [delivered.md](./delivered.md).

Two things follow from that shape. Nothing here tracks status: a registry row says a capability is intended and where it
ranks, never that someone is working on it, so a proposal can sit accepted and unstarted without that being an
oversight. And the registries stay human-owned, which is why an idea has to be argued rather than filed.

[MAINTAINERS.md](./MAINTAINERS.md) documents the promotion workflow in full, along with the commit, formatting, and
delivery conventions this repo is kept to.

## Voice

Documents here are written in one voice, defined in [context/writing-guide.md](./context/writing-guide.md). An issue
does not have to match it. A document you are proposing to add does.
