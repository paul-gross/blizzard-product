# OpenCode hardening specification

## Capability health

A runner capability becomes available only after the configured binary is found, its observed version satisfies the
pinned compatibility range, provider authentication can resolve the configured tier mapping, and the harness selftest
passes. The runner advertises harness id, observed version, and availability state on registration; failure detail stays
runner-local. This describes the runner's own health, and stops there — nothing here explains why a particular chunk is
not moving, which the [harness selection specification](./harness-selection.md) settles as a question the hub does not
answer.

Availability is recalculated at daemon start, after an operator-triggered selftest, and when the observed binary version
changes. Losing availability removes the capability from new acquisition and fresh selection. It does not rewrite
persisted session identity or silently dispatch those sessions elsewhere.

The generic selftest runs the same semantic checks through every adapter: authoritative fresh identity, process exit,
edit and commit, synchronous verdict, automated resume, takeover command, usage parsing, and transcript readability
where the capability claims it. Harness-specific compatibility probes remain additional evidence rather than branches
hidden inside the generic checks.

## Analytics dialect

OpenCode normalized segments register their own analytics dialect. Recognition maps the fixture-proven OpenCode names
and argument keys for file reads, skill invocation, and child-agent spawn into the existing analytics events. The
normalizer version is the dispatch key; unknown versions derive no events rather than guessing from a familiar-looking
tool name.

Child depth and agent type come from normalized sidechains. An unlinked child remains analyzable as child work but does
not acquire a fabricated spawning call. Analytics compare harnesses over Blizzard's event vocabulary while retaining
harness id, harness version, model, and effort as provenance dimensions.

## External subscription usage

A subscription belongs to the operator's provider account, not to a harness binding. One OpenAI plan feeds both OpenCode
and Codex, so a per-adapter sampler would count one window twice or leave it unreported on a night the harness that
owned it never ran. External subscription usage therefore stops being a harness operation: the adapter protocol drops
its sampling method, and the Claude Code binding's Anthropic OAuth sampler moves out of the adapter to become the
provider sampler for an Anthropic subscription.

The runner declares the subscriptions it holds. Each declaration carries a **slug** unique within the runner, which is
the key everything downstream joins on; a human **name** the operator chooses, which is what the board shows and what
may change without breaking a join; the **provider** that selects the sampler; and where that sampler reads its
credentials. An operator with two OpenAI plans describes them as two subscriptions, and it is their slugs, not their
providers, that keep the two apart.

Sampling becomes per subscription throughout. The sample step's cadence is anchored per slug rather than on the newest
sample of any kind, because a single global anchor means sampling one subscription defers every other one past its own
interval — a second declared subscription would simply never be due. Each sample emits one
`external_subscription_usage.sampled` fact carrying its slug, whatever harnesses are configured, available, or working.
A provider with no sampler leaves its subscription declared and unsampled rather than silently absent, and a failed
sample records the attempt without erasing the last good windows. The runner's sample rows gain a slug column; the hub's
record, today one row per runner, becomes one row per runner and slug. The staleness gate ages each subscription on its
own, so one dead sampler cannot blank a healthy one.

The new shape lands **beside** the old rather than replacing it. The runner keeps populating the existing
single-subscription wire field from its Anthropic subscription while also emitting the per-subscription facts, so a hub
and board that have not been upgraded keep working unchanged and no rolling upgrade has a broken window. The board
renders subscriptions from the new per-slug view in a component of its own — the existing panel renders one window list
flat and would collide two subscriptions' identically labelled windows against each other. The old field is retired only
once nothing reads it, which is not this epic's business.

The runner's sampling diagnostic follows its sampler. `blizzard runner external-usage probe` builds a Claude Code
adapter today and calls the very method this epic removes, so it is rebuilt around a declared subscription: it takes a
slug, constructs that subscription's provider sampler, prints the windows it observed, and still writes nothing. The
declared `blizzard:manual-external-usage-probe` verification method is restated the same way — it proves a vendor's live
usage-response shape, which is a claim about a provider rather than about `claude`, and it is the only honest way to
prove a shape no fixture can pin.

The existing configuration migrates to one declared Anthropic subscription, slugged and named by the runner, so an
operator upgrading in place keeps reporting without touching config. Usage remains advisory: nothing in acquisition,
selection, or execution reads a utilization value, so an unsampled subscription never withholds work.

## The operator's view

An epic that opens on what an operator should be able to choose owes the board the two facts that choice creates, and
neither exists there today.

A runner is rendered as one machine with a capacity. It becomes a machine with harnesses: each advertised capability
shown with its harness id, observed version, and availability, so an operator reading the panel can tell a runner that
is idle from a runner that is idle because its OpenCode binding failed its selftest. This is the same panel the
subscription rework already reaches, and the two land together rather than colliding.

Work gains its harness provenance where the model and effort it ran under are already read. An attempt that escalated
overnight is investigated from the chunk's own surfaces, and which harness produced it is part of what the operator is
looking at — the recorded id, not an inference from the model name. Chunk defaults keep their deliberate absence from
the board; this is provenance the fleet reports, not a knob the operator sets.

## Verification layers

| Layer                    | Required proof                                                                                                                                                                                                                                                                                                   |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Adapter unit             | Command composition, tier resolution to model and variant, event parsing, session-id capture from the first record, usage deduplication, subscription-zero cost handling, verdict extraction, permission mapping, and unsupported-feature absence                                                                |
| Fixture contract         | Every pinned OpenCode event, export, child-session, error, and usage shape used by production parsing                                                                                                                                                                                                            |
| Generic selftest         | A real scratch repository through fresh work, commit, judgement, resume, takeover composition, and transcript read, under an identity assertion no harness family fails on shape alone                                                                                                                           |
| Service integration      | A mock OpenCode binary plus mock hub and runner through capability-matched peek for declared sessions, conflicting harness/model preference orders, and a bare lineage with chunk defaults, under both hold and pass-over, claim revalidation, spawn, exit, judgement, usage, transcript pumping, and completion |
| Crash verification       | Kill before session capture, after capture before spawn record, during a tool, between a transcript boundary write and its launch, during resume, and during judgement                                                                                                                                           |
| Shared seam migration    | Two-phase spawn, range-based `read_raw_lines`, and the dropped sampling method leave Claude Code's observable behavior unchanged                                                                                                                                                                                 |
| Mixed-harness acceptance | One runner advertising Claude Code and OpenCode executes a graph whose reachable session requirements need both and preserves correct dispatch across restart                                                                                                                                                    |
| Live provider acceptance | `blizzard:manual-opencode-compatibility` proves the pinned OpenCode version and ChatGPT 5.6 Luna at `max` effort with provider-reported usage and cost                                                                                                                                                           |
| Subscription sampling    | Two declared subscriptions each come due on their own interval and emit one slug-carrying fact independent of configured harnesses; a failed sample preserves the previous windows                                                                                                                               |
| Operator render          | A `web:shell-sweep` or browser e2e render mounts the runner panel and chunk provenance surfaces, proves each harness and subscription remains distinct, and covers narrow width where the surface is mobile-reachable                                                                                            |

## Companion repositories

Capability advertisement and eligibility change the hub-to-runner wire, so the same slices extend `blizzard-mock`. The
mock runner registers configurable harness capabilities and asks to peek under either queue behavior; the mock hub
matches capabilities against the ready order and revalidates claims by the same wire contract; the mock OpenCode harness
supplies root and child sessions, lifecycle events, usage, exports, interruption, and distorted responses through its
lever plane. Wire parity, mock unit tests, service tests, and the fleet end-to-end scenario land with the application
change rather than after it.

The subscription slice extends the companion in the same change. The mock runner emits slug-carrying per-subscription
facts, the mock hub stores and serves more than one subscription per runner without collapsing their windows, and mock
unit and service tests cover independent cadence, a failed sampler preserving earlier windows, and coexistence with the
legacy single-subscription field during the skew window.

The compatibility slice also adds `blizzard:manual-opencode-compatibility` to the application verifiability matrix and
its manual-method detail. Hermetic fixture tests remain the ordinary gate; the named manual method is the repeatable
bridge to a real pinned OpenCode binary and provider account, with credentials excluded from evidence.

## Declared degradations

| Capability                | OpenCode behavior                                                            |
| ------------------------- | ---------------------------------------------------------------------------- |
| Numeric compaction window | Unsupported; omitted rather than translated into reserve tokens              |
| Permission bypass         | Unavailable as an exact analogue; `--auto` still honors explicit denies      |
| Child-to-tool linkage     | Best effort from stable OpenCode identities; otherwise an unlinked sidechain |
| Context-token measurement | Unknown until fixture evidence proves a semantically equivalent value        |

These absences appear in capability diagnostics and are test expectations, not warnings the adapter emits on every turn.
No unavailable optional signal prevents ordinary execution; no feature that participates in correctness may degrade
silently.

## Release gate

An OpenCode version enters the supported range only after the compatibility proof, fixture suite, generic selftest,
service integration, and crash verification pass on that version. A changed external shape either advances the parser
and its normalizer version or keeps the new harness version unavailable. Pinning is the operator's protection from
monthly CLI drift, not a promise that unknown versions probably work.
