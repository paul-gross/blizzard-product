# Migration

Moving in-flight work between immutable workflow graphs on purpose: explicit migration, standing drift, and the
correctness that makes both safe.

| Where                                  | Read when                                                                                          |
| -------------------------------------- | -------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md)       | Reasoning about the initiative as a whole — why immutable graphs need movement machinery           |
| [core.md](./core.md)                   | Looking up the delivered deferred and forced migration, standing drift, retire, and graph metadata |
| [correctness.md](./correctness.md)     | Looking up the delivered transition-keyed derivations and migration-aware fences                   |
| [follow-latest.md](./follow-latest.md) | Looking up the delivered policy of re-pinning in-flight chunks to the newest enabled mint          |
