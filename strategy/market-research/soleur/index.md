# Soleur

Soleur — [github.com/jikig-ai/soleur](https://github.com/jikig-ai/soleur), BSL 1.1, by Jikigai, with a site at
[soleur.ai](https://soleur.ai) — calls itself "the Company-as-a-Service platform": sixty-seven agents across eight
departments, from engineering to legal to sales, offered to a solo founder as the organization they would otherwise have
to hire. It reaches that founder in two forms. One is a very large plugin of skills, agents, and hooks for the coding
harnesses the founder already uses. The other is a hosted web app in which the founder talks to those agents about their
own repository. The plugin is free and available today. The hosted tiers sit behind a waitlist.

It earns a folder for a different reason than [Fusion](../fusion-ai/index.md) did. Fusion is the nearest product to
blizzard. Soleur is the nearest product to the layer *beneath* blizzard: the methodology a worker brings into the
station, which blizzard's [mission](../../../charter/mission.md) deliberately leaves to the workspace and the harness.
Soleur takes that layer, adds a cron-driven way of starting sessions and a chat window on top, and sells the result as
the whole product. Reading it shows what a fleet looks like when it is built entirely out of that layer, with nothing
standing above it. For blizzard, which keeps the two layers apart on purpose, that is the most useful evidence a
competitor could offer.

| File                                                   | When to read                                                                                                                                                |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [capability-comparison.md](./capability-comparison.md) | Weighing a capability against what Soleur ships — the two-way inventory, the ground the products already share, and the decisions the differences ask of us |

## Reading these findings honestly

The products serve different people, and every comparison row should be read with that in mind. Soleur's customer is a
founder running a company, ideally one who does not code. Its own competitive analysis treats engineering fleet
orchestrators as a different customer, worth borrowing ideas from rather than fighting. Blizzard's customer is an
engineer running a fleet. Most of what Soleur has and blizzard lacks therefore falls into the
[Fusion folder's](../fusion-ai/index.md#reading-these-findings-honestly) second kind: a *position difference* the
mission already names. The few genuine gaps are the rows worth reading closely.

Size flatters Soleur more than age flattered Fusion. In eight and a half months, one founder and an agent fleet have
produced 4,267 commits, about 205,000 lines of production TypeScript with half again as much test code, 279 architecture
decision records, 149 post-mortems, and a release on nearly every merge. Its pace is about twenty-three merged pull
requests a day, against blizzard's twelve to twenty-five. The market has barely noticed it. As of 2026-10-09 the
repository has seventeen stars, its Discord has fifteen members, two of a target ten alpha testers are onboarded, and
its own validation verdict is *pivot*, with demand still unproven. That is no judgment on the engineering, which the
[field survey](../factory-field/index.md) placed among the well-structured systems. It does mean nothing in this folder
is market evidence. These findings compare designs, not traction.

One method note, in the same spirit as Fusion's. Soleur's knowledge base is enormous, agent-maintained, and often
contradicts itself. License labels, Stripe's live status, the target customer, and the agent count each differ from one
document to the next, and many decision records are marked *adopting* long after the code went another way. Every claim
below about what Soleur does was taken from code, configuration, or migrations, with the prose used only for intent.
Every claim about what blizzard lacks was checked against shipped source. That check also found two errors in the Fusion
comparison. They are recorded under
[the sandboxing row](./capability-comparison.md#confinement-where-soleurs-hosted-path-is-ahead) and
[shared ground](./capability-comparison.md#where-the-two-products-already-agree) rather than repeated here.

Findings are recorded against Soleur at commit `640ef5a89` (2026-10-09), read from the public repository, its live
GitHub metadata, and soleur.ai, and against blizzard at `0af212d75` (2026-10-08).
