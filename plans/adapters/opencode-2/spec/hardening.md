# OpenCode 2 hardening specification

## Capability health

An OpenCode 2 capability becomes available only when every one of these checks passes, and the runner's health names the
first one that fails:

- the configured binary is found, and its observed version is inside OpenCode 2's admitted range
- the store passes its isolation checks: it is not the operator's personal store and holds no 1.x store
- a private server starts on the store and answers its readiness probe
- the store holds a provider credential that resolves the configured tiers
- every configured variant exists on its model
- the generic selftest passes

The runner advertises harness id, observed version, and availability, as for any harness; failure detail stays
runner-local. OpenCode 1 gains one check of its own: a store OpenCode 2 has migrated makes OpenCode 1 unavailable, by
name.

Availability is recalculated at daemon start, after an operator-triggered selftest, when the observed version changes,
and when a launch-time version comparison refuses. Losing availability removes the capability from new acquisition and
fresh selection.

## Selftest

The generic selftest runs through the OpenCode 2 adapter unchanged, because the driver keeps the seam's shape:
authoritative fresh identity, worker exit, edit and commit, synchronous verdict, automated resume, takeover command,
usage parsing, and transcript readability. `blizzard runner selftest opencode` tests whichever major version the runner
holds. Version-specific evidence stays in the compatibility proof, not in branches inside the generic checks.

## Stopping and process control

Every runner path that stops an OpenCode 2 worker — shutdown drain, pause-park, restart, lease cancellation, forced
takeover — does so with the signals it already sends, and the driver turns the first SIGINT or SIGTERM into an API
interrupt of every running descendant and the root, bounded by a grace, before stopping its server
([execution](./execution.md#stopping-a-turn)). After a group kill the runner confirms no process in the group survives.
No runner path gains a version-specific branch.

## Concurrency

Several drivers' servers share one OpenCode 2 store. The release gate soaks at least four concurrent turns with distinct
permission rules, environments, and operator plugins on one store, and asserts that every turn completes, no session
observes another's configuration, environment, or events, and the store remains readable throughout. If the proof finds
servers cannot safely share a store, OpenCode 2 is blocked until a runner-wide shared-server mode — one server,
isolation through per-session rules and environment — is specified as a successor to this section.

## Analytics dialect

The `opencode-v2` normalizer registers its own hub analytics dialect. Recognition maps the fixture-proven 2.x tool names
and argument keys into the three existing analytics events: `read` and its path argument into `file_read`, the skill
tool into `skill_invocation`, and `subagent` and its agent argument into `agent_spawn`. The normalizer version is the
dispatch key; an unknown version derives no events rather than guessing from a familiar tool name. Harness comparisons
keep harness version as a provenance dimension, from which the major version follows, so OpenCode 1 and 2 are comparable
as one harness and separable when they disagree.

## Declared degradations

Each corpus manifest declares the degradations its version carries, and the operator-facing capability lists them. The
OpenCode 2 binding begins with none declared; any row the proof cannot settle as supported enters the manifest as
degraded or blocks its step, never as an unstated assumption.

## Verification layers

| Layer                    | Required proof                                                                                                                                                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Adapter unit             | Session-id minting, model reference and variant validation, permission composition per autonomy, completion detection, usage and child-usage attribution, verdict extraction from the driver result, error classification |
| Driver unit              | Server launch and readiness, credential handling, environment application, signal-to-interrupt translation, permission and form replies, polling with the event stream lost, and result writing                           |
| Fixture contract         | Every 2.1.4 session, message, part, event, child, permission, form, compaction, error, and usage shape production parsing reads, from the committed corpus                                                                |
| Version dispatch         | Binding selection by observed version, admission by version range, a session the other major version minted reported missing, and the launch-time version comparison                                                      |
| Configuration and store  | One `[opencode]` section driving either major version, the store-isolation refusals, OpenCode 1's migrated-store check, `login` on an OpenCode 2 runner, and every capability health check above                          |
| Transcript source        | Forward reads with re-read of unsettled messages, read-server pooling and its tool-denying configuration, `not_found` against `unreadable`, and interrupted-turn usage recovery                                           |
| Analytics                | The `opencode-v2` dialect over corpus-derived segments, and no events from an unknown normalizer version                                                                                                                  |
| Wire skew                | An OpenCode 2 runner against a hub from before this epic, acquiring and executing unchanged                                                                                                                               |
| Diagnostic service       | The 2.x compatibility diagnostic against the OpenCode 2 CLI-surface artifact, under every lever                                                                                                                           |
| Mock service integration | The real runner against the mock hub and the mock OpenCode 2 through acquisition, mint, prompt, completion, judgement, nudge, takeover, transcript pumping, stopping, and every failure lever below                       |
| Crash verification       | Kill before session create, after create before prompt, during a tool, during a child, between a transcript boundary write and its launch, during resume, and during judgement                                            |
| Mixed-version acceptance | A runner host with an OpenCode 1 runner and an OpenCode 2 runner executes a graph across both, and a runner moved from 1.x to 2.x carries its chunks on in fresh conversations                                            |
| Live provider acceptance | `blizzard:manual-opencode-compatibility` proves 2.1.4 against a real provider, by hand                                                                                                                                    |

Everything above the live row runs hermetically, against the corpus and the mocks, in ordinary test tiers and on any
fleet runner. Only the live row needs OpenCode 2 installed.

## Mock OpenCode 2

OpenCode 2's mock has the same two parts OpenCode 1's has in `blizzard-mock`, kept apart for the same reason: one
executes scripts, the other never does.

**The CLI-surface artifact.** `mock-opencode-2 emit --out <path> --version <v> [--lever NAME]...` bakes a standalone,
stdlib-only fake binary from an OpenCode 2 `opencode_surface` package, sharing no code with OpenCode 1's. It reports the
baked version, answers `serve --stdio --port 0` and the `/api/*` subset with fixed, corpus-shaped responses, and never
executes a caller-supplied prompt. It is what the 2.x diagnostic's service test and rehearsal run against, and it lands
in step 1, before the engine-backed mock exists. Its lever roster follows the rule
`bzh:opencode-lever-roster-extends-both-sides` sets for OpenCode 1: a roster change lands the matching `blizzard` test
fixture in the same change, and the rule is extended to name the OpenCode 2 roster.

**The engine-backed mock server.** An executable, emitted the same way with its server-level levers baked in, whose
`serve --stdio --port 0` answers the API subset the driver uses: session create with a caller-named id, get, update,
list by `parentID`, environment, message list with cursor paging and get by id, prompt with `queue` and `steer`
delivery, inbox list, interrupt, compact, permission requests and replies, form requests and cancellation, the model
listing with variants, and the event stream. Basic auth is enforced against `OPENCODE_PASSWORD`. The server is a
long-lived concurrent process, so it does not run the harness engine in-process: each turn's script runs in a child
process in its own process group and working directory, writing messages through a transcript sink the server reads
live, and an interrupt signals that child's group. A `subagent` call starts a child session whose script runs in a child
process of its own. `opencode --server <url> --session <id>` exists far enough to prove takeover composition.

This is new engine work, not reuse, and the epic owes it: a transcript sink for OpenCode 2, child sessions that run
their own scripts, permission rules enforced against a script's tool calls rather than merely recorded, reasoning
output, and scripted per-step usage and cost in place of generated usage.

Levers reach the mock by one of two mechanisms, each lever assigned to exactly one. Server-level failures are baked into
the emitted executable, because the runner composes the server's command line and passes only an allowlisted
environment: a server that never becomes ready, a dropped event stream, an interrupt the server ignores, a server that
exits mid-turn, a stale outcome left by a dead server, and distorted or unknown response shapes. Turn-level failures are
script helpers: provider refusal, usage limit with reset time, overload, unknown model, unknown variant, a permission
request, a form request, and a child that outlives its root's idle and wakes it.

Both parts are held to the corpus through the wire-parity tier that OpenCode 1 uses: `blizzard-mock`'s `needs_blizzard`
tests feed each part's responses through `blizzard`'s production OpenCode 2 parsers and fail on any shape the parsers
reject, and a corpus for a newly admitted version updates both parts in the same change.

## Release gate

The gate is OpenCode 2's live evidence, run by hand on an operator workstation that has OpenCode 2, because no fleet
runner host carries it. It is the 2.x compatibility proof, the generic selftest, the concurrency soak, and one real
chunk run end to end on a local hub against a 2.x worker — never the hosted hub — with a judgement, a nudge, a takeover,
and an interrupted lease among its turns. OpenCode 1's live proof runs in the same gate, because both major versions
share one runner's composition and a change to it can break either; on every ordinary landing, OpenCode 1 is held by its
hermetic tiers. The gate passes for one admitted version and records it. A runner host joins OpenCode 2 by passing the
same gate on its own machine.
