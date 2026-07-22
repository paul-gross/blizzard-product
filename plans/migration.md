# Plan — `epic:migration`, core slice

> **Shipped.** This plan is frozen: it records what graph migration was conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed. The correctness tail — transition-keyed derivations and fences made migration-aware — is in the [epic registry](../epics.md).

Graphs are immutable and chunks pin the graph they were minted under — that much ships with the engine, and it is what guarantees no chunk is ever stranded at a station its workflow no longer has. But immutability creates the next problem: workflows *do* evolve, and the work in flight when they evolve has to get from the old graph to the new one on purpose. This epic is the movement machinery — migration as an explicit, recorded operation, never a side effect of editing.

## What to build

- **Deferred migration — the default.** The chunk finishes its current node, and re-pins to the target graph at its next transition, landing on the node of the same name. Nothing in flight is interrupted; the swap happens at a boundary, where it is safe by construction. A chunk whose node has no same-named match lands at the target's entry node rather than nowhere.
- **Forced migration — the hotfix path.** For the overnight emergency where waiting for the boundary is the problem: the live lease is revoked by minting a new epoch — the existing fence rejects the abandoned attempt's work — and the current node is redone under the new graph. Abrupt on purpose, and safe for exactly the reason everything else is: the zombie cannot land.
- **Standing drift.** A graph carries `auto_migrate` and a `migration_target` as its operational metadata: publish an edit, point the old graph at it, and in-flight chunks drift over at their own boundaries — one hop per transition, chains resolving step by step so the path self-corrects as pointers change.
- **Retire and re-enable.** Enabling gates exactly one thing: being migrated *into*. A disabled graph strands nothing — its pinned chunks continue, its definition is untouched — it simply stops receiving drifters.
- **The metadata surface.** Enabled, drift intent, and target are the graph's only mutable face, set through the API as appended facts the flags derive from — the definition itself stays immutable forever.
- **A migration is its own fact.** Recorded as what it is, never disguised as a transition — the chunk's history shows where it walked and where it was moved.
