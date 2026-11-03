# OpenCode 2 hardening specification

## Capability health

An OpenCode 2 capability becomes available only when every one of these checks passes, and the runner's health names the
first one that fails:

- the configured binary is found, its version and the service's are in the same major version, and the service's is
  inside OpenCode 2's admitted range
- the operator's managed service is enabled, and is running or starts when the runner runs `opencode service start`
- the service's model listing (`GET /api/model`) resolves the configured tiers through a provider it is authenticated to
  — never `/api/credential`, which returns secret values
- every configured variant exists on its model
- the generic selftest passes

The runner advertises harness id, observed version, and availability, as for any harness; failure detail stays
runner-local. OpenCode 1 gains one check of its own: a store OpenCode 2 has migrated — marked by its `kv` key
`migration.v1-v2` — makes OpenCode 1 unavailable, by name. A probe never swaps the composed binding; a change of major
version waits for a restart ([versions](./versions.md#binding-dispatch)).

Availability is recalculated at daemon start, after an operator-triggered selftest, when the observed version changes,
and when a launch-time version comparison refuses. Losing availability removes the capability from new acquisition and
fresh selection.

## Selftest

The generic selftest runs through the OpenCode 2 adapter unchanged, because the driver keeps the seam's shape:
authoritative fresh identity, worker exit, edit and commit, synchronous verdict, automated resume, takeover command,
usage parsing, and transcript readability. `blizzard runner selftest opencode` tests whichever major version the runner
holds. Version-specific evidence stays in the compatibility proof, not in branches inside the generic checks.

## Stopping

Every runner path that stops an OpenCode 2 worker does so with the signals it already sends. On SIGINT the driver
interrupts the session tree through the API, bounded by a grace; wherever the runner kills a worker instead, or a worker
exits without a result, orphan stops the session tree through the API, best effort, and reconcile catches what that
missed at the next start ([execution](./execution.md#stopping-a-turn)). No runner path gains a version-specific branch,
and none signals an OpenCode process.

## Concurrency

Every turn the runner runs is a session on one service, beside the operator's own. The release gate soaks at least four
concurrent turns with distinct permission rules and environments on the operator's service, while the operator's own
session runs beside them, and asserts that every turn completes and no session observes another's rules, environment,
permission requests, or events.

## Analytics dialect

The `opencode-v2` normalizer registers its own hub analytics dialect. Recognition maps the fixture-proven 2.x tool names
and argument keys into the three existing analytics events: `read` and its path argument into `file_read`, the skill
tool into `skill_invocation`, and `subagent` and its agent argument into `agent_spawn`. The normalizer version is the
dispatch key; an unknown version derives no events rather than guessing from a familiar tool name. Harness comparisons
keep harness version as a provenance dimension, from which the major version follows, so OpenCode 1 and 2 are comparable
as one harness and separable when they disagree.

## Declared degradations

Each corpus manifest declares the degradations its version carries, and the operator-facing capability lists them. The
OpenCode 2 binding begins with two declared, both because a session's settings do not reach its children: a subagent's
permissions follow the operator's agent configuration until the driver applies the session's rules to it, and its tools
run without the lease's environment or identity, which the driver never applies to a child. Any other row the proof
cannot settle as supported enters the manifest as degraded or blocks its step, never as an unstated assumption.

## Verification layers

| Layer                    | Required proof                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Adapter unit             | Session-id minting and runner metadata, model reference and variant validation, the fixed-deny rule list and the reply each autonomy gives, completion detection, usage and child-usage attribution, verdict extraction from the driver result, error classification                                                                                                                                                                                                |
| Driver unit              | Service discovery, environment application to the root alone, signal-to-interrupt translation, permission and form replies, heartbeat from observed tool completions, compaction requested when the lease's window is crossed, polling with the event stream lost, and result writing                                                                                                                                                                               |
| Fixture contract         | Every 2.1.4 session, message, part, event, child, permission, form, compaction, error, and usage shape production parsing reads, from the committed corpus                                                                                                                                                                                                                                                                                                          |
| Version dispatch         | Binding selection by the binary's version, the service's version advertised and recorded, admission by version range, a session without this runner's metadata reported missing, the launch-time comparison, and a change of major version leaving the capability unavailable until restart                                                                                                                                                                         |
| Configuration and store  | One `[opencode]` section driving either major version, `auth_path` and `worker_config_path` ignored on OpenCode 2, OpenCode 1's migrated-store check, and every capability health check above                                                                                                                                                                                                                                                                       |
| Runner on the service    | `opencode service start` run detached under the tool profile, a disabled managed service reported by name, orphan called on every kill and result-less exit except for takeover, reconcile handed only this runner's unfinished, workerless, untaken sessions and finishing before restart resume, and the binding interrupting only what it is handed; both no-ops for Claude Code and OpenCode 1                                                                  |
| Transcript source        | Forward reads with re-read of unsettled messages, raw lines and `tail_position` across root and descendants, reads that never start a turn, a session another runner minted and a foreign cursor refused, `not_found` against `unreadable`, and interrupted-turn usage recovery                                                                                                                                                                                     |
| Analytics                | The `opencode-v2` dialect over corpus-derived segments, the dialect guard iterating both normalizers, and no events from an unknown normalizer version                                                                                                                                                                                                                                                                                                              |
| Wire skew                | No committed contract changes except the additive `drive` and `attend` CLI entries, so a hub from before this epic sees an OpenCode 2 runner as it sees any `opencode` runner                                                                                                                                                                                                                                                                                       |
| Diagnostic service       | The 2.x compatibility diagnostic against the OpenCode 2 CLI-surface artifact, under every lever                                                                                                                                                                                                                                                                                                                                                                     |
| Mock service integration | The real runner against the mock hub and the mock OpenCode 2 through acquisition, mint, prompt, completion, judgement, nudge, takeover, transcript pumping, stopping, and every failure lever below                                                                                                                                                                                                                                                                 |
| Crash verification       | Kill the driver before session create, after create before prompt, during a tool, during a child, between a transcript boundary write and its launch, during resume, and during judgement — each followed by orphan interrupting the session tree — force a takeover mid-turn and assert orphan leaves the attended session alone, kill the runner daemon mid-turn and restart it, asserting reconcile runs before restart resume, and restart the service mid-turn |
| Upgrade acceptance       | A runner stopped on OpenCode 1, upgraded, and started on OpenCode 2 carries its chunks on in fresh conversations, and a runner whose binary changes major version while running reports unavailable until restarted                                                                                                                                                                                                                                                 |
| Live provider acceptance | `blizzard:manual-opencode-compatibility` proves 2.1.4 against a real provider, by hand                                                                                                                                                                                                                                                                                                                                                                              |

Each layer runs under the matrix method that owns its kind: the adapter, driver, fixture contract, version dispatch,
configuration, runner-on-the-service, transcript source, and analytics layers under `blizzard:unit-test`; wire skew
under `blizzard:compat`; the diagnostic service, mock service integration, and upgrade acceptance under
`blizzard:service-test`, upgrade acceptance as a fleet spec under `tests/service/fleet/`; crash verification under
`blizzard:crash-sweep`; and live provider acceptance under `blizzard:manual-opencode-compatibility`.

Everything above the live row runs hermetically, against the corpus and the mocks, in ordinary test tiers and on any
fleet runner. Only the live row needs OpenCode 2 installed.

## Mock OpenCode 2

OpenCode 2's mock has the same two parts OpenCode 1's has in `blizzard-mock`, kept apart for the same reason: one
executes scripts, the other never does.

**The CLI-surface artifact.** `mock-opencode-2 emit --out <path> --version <v> [--lever NAME]...` bakes a standalone,
stdlib-only fake binary from an OpenCode 2 `opencode_surface` package, sharing no code with OpenCode 1's. It reports the
baked version, registers and answers as the managed service with fixed, corpus-shaped responses for the `/api/*` subset,
and never executes a caller-supplied prompt. It is what the 2.x diagnostic's service test and rehearsal run against, and
it lands in step 1, before the engine-backed mock exists. Its lever roster follows the rule
`bzh:opencode-lever-roster-extends-both-sides` sets for OpenCode 1: a roster change lands the matching `blizzard` test
fixture in the same change, and the rule is extended to name the OpenCode 2 roster.

**The engine-backed mock service.** An executable, emitted the same way with its service-level levers baked in, that
registers itself as the managed service under a test's isolated home and answers the API subset the driver uses: session
create with a caller-named id and metadata, get, update, list by `parentID`, environment, message list with cursor
paging and get by id, prompt with `queue` and `steer` delivery, inbox list, interrupt, compact, permission requests and
replies, form requests and cancellation, the model listing with variants, its own version, and the event stream. The
service is a long-lived concurrent process, so it does not run the harness engine in-process: each turn's script runs in
a child process in its own working directory, writing messages through a transcript sink the service reads live, and an
interrupt ends that child. A `subagent` call starts a child session whose script runs in a child process of its own.
`opencode --session <id>` exists far enough to prove takeover composition.

This is new engine work, not reuse, and the epic owes it: a transcript sink for OpenCode 2, child sessions that run
their own scripts, permission rules enforced against a script's tool calls rather than merely recorded, reasoning
output, and scripted per-step usage and cost in place of generated usage.

Levers reach the mock by one of two mechanisms, each lever assigned to exactly one. Service-level failures are baked
into the emitted executable: a service that is never registered or never answers, a dropped event stream, an interrupt
the service ignores, a service that restarts mid-turn, and distorted or unknown response shapes. Turn-level failures are
script helpers: provider refusal, usage limit with reset time, overload, unknown model, unknown variant, a permission
request, a form request, and a child that outlives its root's idle and wakes it.

Both parts are held to the corpus through the wire-parity tier that OpenCode 1 uses: `blizzard-mock`'s `needs_blizzard`
tests feed each part's responses through `blizzard`'s production OpenCode 2 parsers and fail on any shape the parsers
reject, and a corpus for a newly admitted version updates both parts in the same change.

## Release gate

The gate is OpenCode 2's live evidence, run by hand on an operator workstation that has OpenCode 2, because no fleet
runner host carries it. It is the 2.x compatibility proof, the generic selftest, the concurrency soak on the operator's
service, and one real chunk run end to end on a local hub against a 2.x worker — never the hosted hub — with a
judgement, a nudge, a takeover, and an interrupted lease among its turns. OpenCode 1 is held by its hermetic tiers, and
its live proof runs on a machine that has OpenCode 1, because the two major versions do not share a machine. The gate
passes for one admitted version and records it. A runner host joins OpenCode 2 by passing the same gate on its own
machine.
