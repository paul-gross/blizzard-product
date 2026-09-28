# Resolution tracing contract

## One resolution, traced

A session's model and effort are decided in three places today, each returning a bare value:

1. **The hub merge.** `EffectiveSession.of(chunk, graph, node)` combines a declared session with the chunk's defaults,
   field by field, and stamps the result onto the node envelope as `session_model`, `session_effort`, and
   `session_harnesses`. The runner never sees which layer supplied a field.
2. **Harness selection.** `HarnessSelector.select(node)` walks the acceptable harness set in order and returns the first
   member that can serve the session, with the members it skipped.
3. **Adapter resolution.** Each binding's `resolve_model`, `resolve_model_strict`, and `resolve_effort`, over the shared
   left-to-right skeleton in `harness_shared`, turn authored strings into what reaches the harness process.

The report is built from these same three phases, never from a copy of their rules. Each phase gains a traced form that
returns the value together with the layer that supplied it. The existing entry points become projections of the traced
form, so the spawn path and the report cannot disagree: `Spawner` keeps calling `SessionResolver.session_stamps`, which
keeps its signature and takes `.value` from the traced result. A test pins the agreement: for every configuration
fixture and authored session in the suite, the traced value equals what `session_stamps` returns for a fresh mint.

## Provenance vocabulary

Every resolved cell carries one source from a closed set. The JSON uses the kebab-case term; the terminal renders it as
words.

| Source             | Applies to            | Meaning                                                                                            |
| ------------------ | --------------------- | -------------------------------------------------------------------------------------------------- |
| `graph`            | session field         | The declared session supplied the field                                                            |
| `chunk`            | session field         | The declaration left the field empty, and the chunk's default supplied it                          |
| `unset`            | session field, effort | Neither layer supplied it                                                                          |
| `runner-alias`     | model, effort         | An entry in this runner's alias table for this harness                                             |
| `adapter-default`  | model                 | The binding's built-in tier table                                                                  |
| `native`           | model                 | An unprefixed, harness-native model name passed through as authored                                |
| `adapter-fallback` | model                 | Nothing in the preference resolved, and the non-strict path fell back to the binding's own default |
| `unmapped`         | tier                  | Neither the runner nor the binding maps this tier for this harness                                 |
| `as-authored`      | effort                | A well-known effort passed through unchanged                                                       |
| `dropped`          | effort                | An effort the binding does not recognize; nothing reaches the harness                              |

## The hub merge

`EffectiveSession` records a source for `model`, `effort`, and `harnesses` beside each value. The rule is the one
already pinned: a declaration's `model` or `harnesses` list outranks the chunk default whenever it is nonempty, as a
whole list and never merged, and its `effort` outranks the chunk default whenever it is not null. A node with no
declared session — a bare `fresh`, `resume`, or `resume:<node>` lineage — takes all three from the chunk.

The merge is factored so that a declared session can be resolved without a node: `EffectiveSession.of` delegates to a
declaration-level merge over `(declaration | None, chunk defaults)`, and the report calls that same merge, with empty
defaults when no chunk is named. The node envelope does not change shape; sources travel only on the read below.

## The hub read

The runner asks the hub for merged sessions on two new routes under `/api/fleet`, behind `require_runner_principal` like
the rest of the fleet router. A runner token is enough; no lease is needed.

- `GET /api/fleet/sessions?graph=<name>` answers for the effective graph of each name — the newest graph that is not
  retired — or only for the named one. Chunk defaults are empty, so every field's source is `graph` or `unset`.
- `GET /api/fleet/chunks/{chunk_id}/sessions` answers for the chunk's pinned graph merged with that chunk's defaults.
  When the pinned graph has a runner-executed node on a bare lineage, the response adds one entry with a null session
  name for it.

Each response entry carries the graph's id and name, the session name in authored order, and for each of the three
fields the declaration's value, the chunk's value, the effective value, and its source. The wire model lives in
`blizzard.wire` beside `GraphSessionView`. An older hub answers `404` on these routes; the runner treats that the same
way as an unreachable hub (see [command.md](./command.md)) and says which it was.

## Adapter resolution

Model resolution is traced for each preference entry, left to right, for one harness:

- **A tier** (`blizzard:`-prefixed). The runner's alias table for this harness is consulted first — `[models.aliases]`
  for Claude Code, `[opencode.models.aliases]` for OpenCode — and gives `runner-alias`. Then the binding's built-in tier
  table gives `adapter-default`. Claude Code's table maps `blizzard:frontier` to `fable`, `blizzard:advanced` to `opus`,
  and `blizzard:basic` to `sonnet`; OpenCode's table is empty. Otherwise the entry is `unmapped`, and the next entry is
  tried.
- **A harness-native name** (unprefixed). It passes through as the binding already passes it: `native`, or
  `runner-alias` where the binding renames it through its alias table. OpenCode accepts only a name that parses as a
  `provider/model` reference; any other name is unresolved for OpenCode.
- **Nothing resolved.** The strict path returns no model, which is what makes a harness unable to serve the session. The
  non-strict path falls back to the binding's default: Claude Code's `claude-opus-5`, or, for OpenCode, no `--model` at
  all, leaving OpenCode's own configured model to decide. Either way the source is `adapter-fallback`.

The trace keeps the winning entry (`via`) and each earlier entry with the reason it did not resolve.

Effort is traced per harness. The runner's alias table — `[effort.aliases]` for Claude Code, `[opencode.effort.aliases]`
for OpenCode — gives `runner-alias`. Otherwise one of `low`, `medium`, `high`, or `max` passes through `as-authored`.
Any other value is `dropped`: the binding logs it once and passes nothing, so the harness runs at its own default. A
value that reaches the harness is rendered as the argument it becomes: `--effort` for Claude Code, and `--variant` for
OpenCode.

## Harness selection

The session's effective harness set is walked exactly as `HarnessSelector.select` walks it, under the same strict gate:
selection is strict when the model preference is nonempty and either the set has more than one member or the preference
names a tier. The per-member check is extracted from `select` so the report can apply it to members after the winner
too. Each member of the set receives exactly one outcome:

| Outcome       | Meaning                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| `selected`    | The first member that passes; the session would be minted here                                                |
| `passed-over` | A later member that would also pass, shown with what it would run                                             |
| `skipped`     | A member that fails, with the selector's reason: `unknown`, `unavailable`, `unhealthy`, or `no-authored-tier` |

When the effective set is empty, the runner's default harness is the only candidate, and the pool records that the
default, not an acceptable set, admitted it. When every member is skipped, there is no selection: the runner would
escalate without spawning, and hub eligibility would not have offered it the chunk.

## Pools already running

A resume never re-resolves. It replays the `resolved_model` and `resolved_effort` recorded on the lease that minted the
session, so an alias edited after a pool was minted changes the next fresh session and leaves the current one alone.
When a chunk is named, the report reads this runner's own store for the pool head of each session in that chunk and
shows its recorded harness, model, and effort beside the fresh-mint answer, flagging where they differ. The read goes
through the existing lease-session repository; the report opens no write path to the store.

## Proof

- The agreement test above, run over fixtures that exercise every source in the vocabulary.
- Route tests: a runner token without a lease is answered; a request without a runner token is refused; an unknown graph
  or chunk returns `404` with the name that did not resolve.
- A merge test per field, showing the source changes from `graph` to `chunk` exactly when the declaration's value is
  empty or null, alongside the existing test pinning the precedence itself.
