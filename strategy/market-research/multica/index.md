# Multica

Multica — [github.com/multica-ai/multica](https://github.com/multica-ai/multica), under the source-available Multica
License, by Index Labs (Hong Kong), with a hosted product at [multica.ai](https://multica.ai) — calls itself "project
management for human + agent teams." Its promise is that an agent is a teammate: it shows up on the board with a name,
gets assigned an issue the way a colleague would, comments as it works, raises a blocker when it is stuck, and hands the
result back for review. A daemon on any machine the team controls drives whichever agent CLI is installed there, from a
list of twenty-six, and the same workspace is reachable from a web app, a desktop app, an iPhone build, a CLI, and five
chat platforms.

It earns a folder because it is the product the market has chosen. [Fusion](../fusion-ai/index.md) is the nearest design
to blizzard, and [Soleur](../soleur/index.md) the nearest to the methodology layer beneath it. Multica is the nearest to
blizzard's *surface*, the board where a human watches agents work, and it has found an audience on a scale nothing else
in the [field survey](../factory-field/index.md) approaches. It takes the opposite answer to the question blizzard's
[mission](../../../charter/mission.md) settles first. Blizzard refuses to be the project-management system and treats
the backlog as a seam a tracker plugs into. Multica is the tracker, and the fleet is a feature of it. Reading it shows
what a team gets when the agents are added to the place where work is already tracked, and what it gives up by having
nothing beneath the board but the agents' own good behaviour.

| File                                                   | When to read                                                                                                                                                 |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [capability-comparison.md](./capability-comparison.md) | Weighing a capability against what Multica ships — the two-way inventory, the ground the products already share, and the decisions the differences ask of us |

## Reading these findings honestly

The products are not for the same person, though they look it from a distance. Multica's customer is a small team,
humans first, that wants to hire agents into the way it already works: "two engineers and a fleet of agents can move
like twenty," in its own vision's words. Blizzard's customer is an engineer who queues work, walks away, and wants to
come back to landed work. A Multica feature blizzard lacks is therefore read the way the
[Fusion folder](../fusion-ai/index.md#reading-these-findings-honestly) set out: a *schedule difference* blizzard has
already placed in [`epics.md`](../../../epics.md), a *position difference* its mission already takes, or a *genuine gap*
that has not yet had a decision.

Unlike Soleur, Multica's numbers are market evidence. As of 2026-10-29 the repository has 54,220 stars and 7,136 forks
after nine and a half months, 369 contributors, 168 releases, and 1,846 open issues and pull requests. Its own team is
real and human, with three engineers at about a thousand commits each, and its fleet is real too. In the thirty days to
2026-10-29, 204 of 280 commits on `main` carried the trailer of its own agent account, and that agent merged half of the
284 pull requests that landed, most without a GitHub review. Its pace, about nine and a half merged pull requests a day,
is half blizzard's nineteen, and has fallen steadily since April. Its popularity says the shape of the product is what
buyers want. It says nothing about how well the shape holds up overnight, which is the question this folder answers from
the code.

One method note. Multica's documentation is candid and mostly accurate, and its security-model page in particular says
plainly what the code does. Where docs and code disagree, it is usually in Multica's disfavour by a small margin: a
review gate that the docs present as a promise is, in code, an instruction to the agent. Every claim below about what
Multica does was taken from code and migrations, with the prose used only for intent. Every claim about what blizzard
lacks was checked against shipped source.

Findings are recorded against Multica at commit `2083970b5` (2026-10-29), read from the public repository, its live
GitHub metadata, and multica.ai, and against blizzard at `088437357` (2026-10-29).
