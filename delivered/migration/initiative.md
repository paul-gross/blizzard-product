---
initiative: migration
refinement: pristine
epics:
  - core
  - correctness
  - follow-latest
---

# Plan — `initiative:migration`

> **Shipped.** This plan is frozen: it records what graph migration was conceived to be, and its entry in
> [delivered.md](../../delivered.md) is the record of what landed. The correctness tail — transition-keyed derivations
> and fences made migration-aware — is in the [epic registry](../../epics.md).

Graphs are immutable and chunks pin the graph they were minted under — that much ships with the engine, and it is what
guarantees no chunk is ever stranded at a station its workflow no longer has. But immutability creates the next problem:
workflows *do* evolve, and the work in flight when they evolve has to get from the old graph to the new one on purpose.
This epic is the movement machinery — migration as an explicit, recorded operation, never a side effect of editing.
