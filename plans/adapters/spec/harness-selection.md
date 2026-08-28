# Harness selection specification

Harness selection is deterministic orchestration above the adapter seam. Adapters translate operations for one harness;
they never choose whether they should receive an operation.

## Vocabulary

A **harness id** is an opaque, stable identifier for one adapter family. The initial ids are `claude_code` and
`opencode`; `codex` is the next intended occupant. Renaming an id breaks persisted session dispatch and authored graph
constraints.

A **runner capability** is one enabled harness id together with the observed harness version and its availability
evidence. Configured but missing, version-incompatible, or failed-selftest bindings remain visible diagnostically but do
not satisfy acquisition. One available capability is the runner's default; a duplicate id or a default that is not
available is a configuration error.

An **acceptable harness set** is an ordered, nonempty list of harness ids. Order is preference and membership is
permission: the first available member wins, and no harness outside the list may substitute. This deliberately differs
from a model preference, which never fails a spawn: an unresolvable one falls back to the adapter's own default model.

A **session reference** is the pair `(harness_id, session_id)`. Session ids are opaque within the namespace of the
harness that minted them; Blizzard never assumes two harness families cannot produce the same text.

## Authoring and inheritance

A declared graph session may carry `harnesses`, an acceptable harness set beside its model, effort, compaction, and
rotation policy. Harness-specific nodes use distinct declared sessions; a node does not carry an independent harness
field because two nodes resuming one session cannot choose different owners for it.

The chunk carries `default_harnesses` beside its model and effort defaults. It is editable only while the chunk is
unclaimed and is empty when the operator expresses no constraint. A declared session's `harnesses` outranks the chunk
default as a whole; these lists are not merged or reordered.

Both fields are authored through surfaces that must move together, and they are listed here because a field added to a
model and forgotten on the way to a person is the ordinary way this work ships half-done. `harnesses` reaches the graph
through the session declaration, its validation, the graph-session table and its migration, and the graph read view.
`default_harnesses` reaches a chunk through the chunk-edit path — which writes the existing defaults as one atomic set,
so a third field joins that write rather than sitting beside it — the chunk request and read models, and the hub CLI's
chunk-set command.

`default_harnesses` takes no board surface, following the precedent its siblings set: `default_model` and
`default_effort` are deliberately CLI-authored and read back with `chunk show`, because the board once offered a model
edit that changed nothing about what the fleet ran and the lesson was to stop offering knobs the envelope ignores.
Neither field is designed here; both are named so no slice can land believing it is finished.

The acceptable set rides the node envelope, exactly as model and effort already do. The hub resolves the declared
session's `harnesses` against the chunk's `default_harnesses` — the declared list outranking the chunk default as a
whole — and stamps the winner onto the envelope it hands the runner. One resolution, computed where the graph and the
chunk both live.

The runner obeys it. It selects the first member its available capability registry supports, and consults its own
default harness only when the envelope carries no set at all. It never re-derives the levels above it, so a runner and
its hub cannot disagree about what a node was allowed to run. A present graph list must be nonempty and unique; the
graph mint validates its shape but not current fleet availability, and an unsupported but well-formed graph remains
valid and unclaimable until a matching runner exists.

Bare `fresh`, bare `resume`, and node-addressed `resume:<node>` forms use the chunk and runner levels because they
reference no declared session. When any resume form has no stored session and therefore falls back to fresh, that mint
follows the same resolution order.

Acquisition eligibility gates on declared sessions only, and this is sound rather than a shortcut: a lineage that names
no declared session carries no harness constraint to violate, and a lineage that does is visible to the walk while it is
still reachable. The one lineage that changes harness across a runner boundary is a `resume:<node>` whose stored session
lives on a runner that no longer holds the chunk — the fresh mint that follows resolves under the new holder, exactly as
the lost conversation itself does.

## Capability tiers

A graph asks for a **capability tier** — `frontier`, `advanced`, and the rest of Blizzard's tier vocabulary — not for a
provider's model string. The tier is the only model language a graph author writes, because a lineage that may run on
either harness cannot name a model both harnesses have.

Each configured harness resolves a tier to its own concrete pair of model and effort. One runner's `frontier` may be a
Claude model at `high` through the Claude Code binding and an OpenAI model at `max` through OpenCode; both satisfy the
same authored tier. The mapping is runner configuration, declared per harness, and a tier with no mapping for a harness
makes that harness unable to satisfy a session demanding it.

This is what keeps harness order and model intent from contradicting each other. Selection resolves the harness first,
then asks that harness's mapping for the tier; a session that could only ever run one provider's model expresses that by
constraining `harnesses`, never by naming the model. An explicitly authored model string remains legal for a
single-harness lineage and is passed through to the adapter unchanged, where an unresolvable value still falls back to
the adapter default rather than failing the spawn.

## Session identity

The selected harness id is immutable identity of the concrete session and of the declared pool while that pool remains
on the runner. Every operation against that session — resume, judgement, answer delivery, nudge, context read,
transcript read, and takeover — dispatches directly through the recorded adapter. None re-resolves graph, chunk, or
runner defaults.

Every persisted lookup and uniqueness rule that addresses a session uses the full session reference. Lease history, pool
heads, preamble fingerprints, transcript cursors, usage recovery boundaries, takeovers, escalations, and runner API
reads may render the raw session id for a person, but they resolve it together with its harness id. A raw id without an
owner is not a dispatch key.

A rotation breach creates a replacement session in the same pool and retains the pool's harness. A pool newly
established after reassignment or graph migration resolves afresh because pools do not cross runners or graph pins. A
graph moving from one harness to another does so by entering a different declared session; the sessions share the
chunk's durable artifacts and the environments under current tenure, not conversational context.

Within-node retries mint fresh sessions but retain the prior attempt's harness in the first breadth slice. If that
capability is unavailable or no longer permitted, the attempt escalates explicitly. Cross-harness retry may later choose
another member of the authored acceptable set only under a separate policy that records the switch; ordinary failure
never advances the list implicitly.

## Existing sessions

Every session id recorded before this epic was minted by Claude Code, so the migration is a single stamp rather than a
reconciliation. The runner-store migration adds the harness id beside the session id on the six tables that carry one —
`leases`, `asks`, `takeovers`, `session_preamble_facts`, `context_samples`, and `transcript_segments` — and backfills
`claude_code`. The hub's `questions.session_id` receives the same stamp. A session pool needs no migration because a
pool head is derived from the leases themselves and inherits their stamp for free.

Existing rows have no trustworthy mint-time Claude Code version, so that version remains unknown; the configured Claude
Code adapter may still resume them, and the first new invocation records the actual version it used without rewriting
earlier history. Legacy operator and API surfaces continue to display the raw id, while internal dispatch and any new
wire shape carry the owner. Migration verification covers resume, judgement, transcript read, and takeover of a
pre-migration Claude Code session.

## Acquisition

Sticky tenure requires a runner to be able to carry the chunk through the graph rather than merely start its current
node. The hub owns one eligibility calculation over the pinned graph, chunk defaults, and the runner capability
snapshot. From the chunk's current position, it computes every statically reachable runner-executed declared session.
The runner is eligible only when each session's effective authored set intersects its available capabilities.
Hub-executed nodes are excluded; branches and cycles are included once; different sessions may be satisfied by different
capabilities on the same runner.

The fleet's queue read separates from the operator's. Until now `GET /api/fleet/queue/peek` returned the operator
queue's own rendering, which suited a hub that had nothing to say about who was asking. A capability-matched peek does,
so the two routes diverge: the operator route keeps rendering the fleet-wide ready order, and the fleet route becomes a
request in its own right.

Fleet peek requires runner authentication. Eligibility is answered for the authenticated runner and no other, so a
caller can never ask what some other runner is allowed to claim, and a hub that cannot identify its caller returns no
entry rather than an unfiltered queue.

The request carries what the hub needs to match and nothing more: the runner's available capabilities, and whether it
wants to hold at the first entry it cannot work or pass over it. The hub walks the ready order, applies the eligibility
calculation to each entry, and returns the first match — or, under hold, returns nothing once it reaches an entry the
runner cannot work. Holding suits a machine bought to run one harness's work; passing over suits a mixed fleet, and is
the default so a single unclaimable chunk cannot stall one by accident.

The hub explains nothing about a skip. It returns an entry or it returns none; it records no reason, derives no
fleet-wide blocked diagnostic, and gives an unclaimable chunk no second status. A chunk no runner can carry keeps its
ready position and waits. Why work is not moving is an operator-facing question this slice deliberately leaves
unanswered.

A runner one minor version behind its hub keeps working, because none of this is expressed as a break. The project's
skew window already promises that a hub serves the previous minor's runners, and the queue change is designed to sit
inside it rather than to need a new mechanism: the capability set and the hold-or-pass-over preference are optional
request fields, and the entries the fleet route returns keep the field names the previous shape used. A runner that
sends neither field is answered exactly as it is today — the unfiltered ready order, head first — because a hub that was
told nothing about a runner's capabilities filters on nothing.

That silence carries through to the claim. A runner that never asserted capabilities is not revalidated against them, so
it can never meet the incompatibility denial it has no branch for. An upgraded runner opts into filtering and into the
denial in the same act, which is what keeps a versioned endpoint unnecessary here: the change is additive in both
directions, and the one behavior that could surprise an old client is reached only by clients that asked for it.

Route claim repeats the eligibility calculation while installing the route, against the capabilities the runner
registered. A capability that changed between peek and claim therefore returns a distinct incompatibility denial; the
runner releases the environments it acquired and continues filling from a fresh peek.

A bare lineage introduces no graph-specific requirement unless the chunk has `default_harnesses`; otherwise any runner
with a valid default can satisfy it.

An eager cross-graph restart preserves tenure, so it validates the target graph's reachable requirements against the
holding runner before changing the pin. A migration that releases tenure relies on normal acquisition. No session pool
survives either graph change, even where declaration names coincide.

Capability loss after claim has two outcomes. An operation on an existing recorded session escalates in place because no
other runner can resume it. Before minting a fresh session for a later node, the holding runner gives tenure back
without consuming the node's retry when another runner may satisfy the requirement; ordinary acquisition then decides,
and the chunk waits in the queue when no runner is eligible. An eager restart that cannot be satisfied is refused before
mutation.

## Provenance and failure

Capabilities travel on runner registration, beside the environment capacity it already carries, and re-register when
availability changes. They are deliberately not facts: the fact lane is buffered and forwarded, and eligibility read
through it would be stale by as long as the buffer is deep. Registration is synchronous, so the record claim validates
against is the one the runner last asserted.

The session records harness id and mint-time version. Every runner-executed lease generation records the actual id and
version invoked; every transcript segment records producing harness id and version beside its normalizer version.
Escalations and takeover surfaces report the recorded identity. Hub-executed nodes carry no coding harness.

Loss of a capability after acquisition never causes substitution inside an existing session. The runner may finish work
already alive under it, but any operation that cannot dispatch to the recorded adapter becomes a specific blocked or
escalated condition. Unknown persisted ids, incompatible resumes, and unsupported fresh requirements are diagnosable
states, never fallback to the runner default.
