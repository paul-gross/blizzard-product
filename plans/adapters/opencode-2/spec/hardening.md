# OpenCode 2 hardening specification

## Capability health

A line-2 capability becomes available only when the configured binary exists, its observed version falls in the line-2
admitted range with a committed corpus inside it, the store passes its isolation checks, a private server starts and
answers its readiness probe, the store holds a provider credential that resolves the configured tiers, the configured
variants exist on their models, and the generic selftest passes. The runner advertises harness id, line, observed
version, and availability; failure detail stays runner-local and names the specific failed check.

Availability is recalculated at daemon start, after an operator-triggered selftest, when the observed version changes,
and when a launch-time version check refuses. Losing availability removes the line from new acquisition and fresh
selection. It does not rewrite any session's recorded line or dispatch its sessions to the other line.

## Selftest

The generic selftest runs through the line-2 adapter unchanged: authoritative fresh identity, completion, edit and
commit, synchronous verdict, automated resume on a second server, takeover command, usage parsing, and transcript
readability. Line-specific evidence stays in the compatibility proof, not in branches inside the generic checks.

## Interruption and process control

Every runner path that ends a line-2 turn early — restart, forced takeover, lease cancellation, shutdown drain —
interrupts the root session through the API, waits a bounded grace for it and its descendants to settle, and then kills
the server's process group. A server that does not answer the interrupt is killed without waiting. After the kill the
runner confirms no process in the group survives. The session's next read reports `interrupted` or `failed`; a session
the API still reports as running after its server is gone is a defect the proof row guards against.

## Concurrency

Several lease servers share one line-2 store. The release gate soaks at least four concurrent leases with distinct
permission rules, environments, and operator plugins on one store, and asserts that every turn completes, no session
observes another's configuration, environment, or events, and the store remains readable throughout. If the proof finds
servers cannot safely share a store, the line is blocking until a runner-wide shared-server mode — one server, isolation
through per-session rules and environment — is specified as a successor to this section.

## Analytics dialect

The `opencode-v2` normalizer registers its own hub analytics dialect. Recognition maps the fixture-proven 2.x names —
`read` and its path argument, the skill tool, `subagent` and its agent argument, `shell` — into the existing analytics
events. The normalizer version is the dispatch key; an unknown version derives no events rather than guessing from a
familiar tool name. Harness comparisons keep line and version as provenance dimensions, so OpenCode 1 and 2 are
comparable as two lines of one harness and separable when they disagree.

## Declared degradations

Each corpus manifest declares the degradations its version carries, and the operator-facing capability lists them. The
line-2 binding begins with none declared; any row the proof cannot settle as supported enters the manifest as degraded
or blocks its step, never as an unstated assumption.

## Verification layers

| Layer                    | Required proof                                                                                                                                                                                                          |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Adapter unit             | Session-id minting, model and variant reference composition and validation, permission rule composition per autonomy, completion detection, usage and child-usage attribution, verdict extraction, error classification |
| Fixture contract         | Every 2.1.4 session, message, part, event, child, permission, compaction, error, and usage shape production parsing reads, from the committed corpus                                                                    |
| Line dispatch            | Version-to-line admission, the `@line` selector at graph mint and acquisition, a session's line surviving restart, refusal of a binary from the wrong line, and the launch-time version recheck                         |
| Mock service integration | The mock OpenCode 2 plus mock hub and runner through acquisition, mint, prompt, completion, judgement, nudge, takeover, transcript pumping, interruption, and every failure lever below                                 |
| Crash verification       | Kill before session create, after create before prompt, during a tool, during a child, between a transcript boundary write and its launch, during resume, and during judgement                                          |
| Mixed-line acceptance    | One runner advertising Claude Code, OpenCode 1, and OpenCode 2 executes a graph whose sessions need all three, including a bare `opencode` and an `opencode@1`, and dispatches each correctly across restart            |
| Live provider acceptance | `blizzard:manual-opencode-compatibility` proves 2.1.4 against a real provider, by hand                                                                                                                                  |

Everything above the live row runs hermetically, against the corpus and the mock, in ordinary test tiers and on any
fleet runner. Only the live row needs OpenCode 2 installed.

## Mock OpenCode 2

`blizzard-mock` gains a line-2 OpenCode surface beside its line-1 surface, sharing the mock harness engine — the prompt
is the program, executed in the worktree with the session's environment — and nothing of line 1's shapes. It is a
separate executable reporting `opencode v2.1.4` by default, with the version a lever, so line dispatch, wrong-line
refusal, and an upgrade beneath a running runner are testable.

Its `serve --hostname 127.0.0.1 --port 0` answers the stable `/api/*` subset the binding uses: session create with a
caller-named id, get, list by `parentID`, message list with cursor paging, prompt with `queue` and `steer` delivery,
interrupt, compact, environment, permission rules, permission requests and replies, the model listing with variants, and
the event stream with per-session `seq`. Basic auth is enforced with the credential the proof establishes. A behavior
script can edit the worktree, run a shell tool under the session's permission rules, spawn a `subagent` child session
that runs its own script, raise a permission request and wait for its reply, and emit text, reasoning, `<Choice>`, and
per-step usage and cost. `opencode --standalone --session <id>` exists far enough to prove takeover composition.

Every response shape the mock emits is taken from, and tested against, the committed 2.1.4 corpus, so the mock and the
real binary answer to the same evidence; a corpus for a new admitted version updates the mock in the same change.

Levers inject the failures the binding must survive: provider refusal, usage limit with reset time, overload, unknown
model, unknown variant, a server that never becomes ready, a dropped event stream, an interrupt the server ignores, a
server that exits mid-turn, a child that outlives its root's idle, a session the API still reports running after its
server is gone, and distorted or unknown response shapes.

The mock runner advertises line-carrying capabilities and the mock hub matches `@line` selectors by the same wire
contract as the real ones, with wire parity, mock unit tests, and service tests landing with the application change
rather than after it.

## Release gate

The gate is the line's live evidence, run by hand on an operator workstation that has OpenCode 2, because no fleet
runner host carries it. It is the 2.x compatibility proof, the generic selftest, the concurrency soak, and one real
chunk run end to end on a local hub against a 2.x worker — never the hosted hub — with a judgement, a nudge, a takeover,
and an interrupted lease among its turns. The gate passes for one admitted version and records it; a later version
inside the range is admitted only by re-running the proof and committing its corpus. A runner host joins line 2 by
passing the same gate on its own machine.

Line 1 is regression-verified by the same gate's 1.x half on every landing, because both lines live in one runner and a
change to shared composition can break either.
