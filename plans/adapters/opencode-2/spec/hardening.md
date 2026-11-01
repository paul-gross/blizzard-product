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

## Release gate

Line 2 is built and verified by hand on an operator workstation that has OpenCode 2, because no fleet runner host
carries it. The gate is the 2.x compatibility proof, the generic selftest, the concurrency soak, and one real chunk run
end to end on a local hub against a 2.x worker — never the hosted hub — with a judgement, a nudge, a takeover, and an
interrupted lease among its turns. The gate passes for one admitted version and records it; a later version inside the
range is admitted only by re-running the proof and committing its corpus. A runner host joins line 2 by passing the same
gate on its own machine.

Line 1 is regression-verified by the same gate's 1.x half on every landing, because both lines live in one runner and a
change to shared composition can break either.
