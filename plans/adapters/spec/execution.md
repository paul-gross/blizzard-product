# OpenCode execution specification

## Composition

Runner configuration builds a registry of harness adapters and transcript sources keyed by stable harness id. The
reconciliation loop, hosted runner API, selftest service, takeover service, transcript usage recovery, and diagnostics
receive the same registry through one composition path. Claude Code-specific construction does not remain duplicated at
those roots.

The OpenCode adapter returns harness-neutral values, and keeps model and effort mapping, output parsing, permission
translation, lifecycle integration, usage translation, and transcript access inside the binding. Harness selection and
retry policy remain outside it.

Two operations on the shared seam change shape for every harness, and this slice owns both rather than treating them as
OpenCode-local. `spawn` becomes two phases so a harness that mints its own session id can be launched before its
identity is known, and the seam sheds its subscription-sampling method entirely. The Claude Code binding is migrated to
both: it keeps passing its generated `--session-id` and simply returns an identity that is already known at phase two,
so its observable behavior does not change. A seam change with one implementation is how the first adapter's shape
becomes everyone's shape by accident, which is the failure this epic exists to end.

## Worker process

An unattended OpenCode turn runs as `opencode run --format json` in the leased workspace. Resume adds the recorded
`--session`; fresh mint adds the resolved `--model`; resolved effort maps to `--variant`; unattended permission policy
maps to `--auto` plus runner-owned explicit rules. Judgement and nudge use the same resumed command synchronously.
Interactive takeover composes the OpenCode TUI command for the recorded session and workdir.

The per-turn `opencode run` process is the lease PID. Blizzard records its process start time, treats its exit as the
turn boundary, and kills it before any automated restart or forced takeover. A shared `opencode serve` process is not
the worker: one shared PID cannot represent one lease, and tool subprocesses execute in the server's environment rather
than the attached client's per-lease environment.

## Fresh-session handshake

OpenCode assigns fresh session ids, so launch and identity registration are two durable phases. The adapter first
launches the worker in its own process group and returns a pending handle carrying PID and process start time before
waiting for OpenCode. The spawner records that provisional process against the lease, then asks the handle to await
identity with a bounded startup timeout.

Identity arrives on the worker's own stdout. Every record `opencode run --format json` emits carries the root
`sessionID`, beginning with the first, so the adapter reads the id from the stream it already parses for output and
usage. No plugin, no side channel, and no second process participates in learning who the worker is. Once the id is
read, the spawner records the authoritative session reference and marks the generation identified.

The child installs a parent-death signal before exec, closing the smaller launch-before-provisional-record window: if
the daemon dies there, the operating system kills the worker group. Once a provisional PID is durable, startup recovery
can identify the pending generation. It kills the recorded process group and fails that unidentifiable generation before
retrying; an OpenCode session created but not durably identified may remain as inert harness history, but no process
survives to mutate the worktree. If the session reference is durable, ordinary restart recovery resumes it through the
recorded adapter.

Timeout, malformed identity, or process exit before the first record kills the whole process group and records a spawn
failure; no empty or hinted id is recorded as success. The crash-point registry brackets launch, provisional process
recording, identity arrival, and identified-session recording, and the invariant checker asserts that no active lease
has an unowned live process after recovery.

The generic adapter selftest has to be relaxed in this slice rather than the hardening one, because until it is,
OpenCode cannot pass it and an unpassed selftest is what keeps a capability unavailable. Its spawn check currently fails
the run unless the returned session id equals the hint the runner supplied — an assertion only a harness that accepts
imposed ids can satisfy. It becomes the weaker, harness-neutral claim the seam actually makes: the returned id is
non-empty and authoritative, and equals the hint only where the adapter declares it honors one. Claude Code keeps
passing on the stronger reading; OpenCode passes on the honest one.

Resume never performs the handshake because the stored session reference is already authoritative, but every resumed
process still receives the same process-group and parent-death ownership.

## Runner-owned plugin

The runtime scaffolds an OpenCode plugin in runner-owned configuration, never in a project repository. With identity
arriving on stdout, the plugin is left with the two jobs no other channel can do. It inherits the allowlisted worker
environment and:

- After every completed tool call, including child-session tools, invokes `blizzard runner heartbeat` softly.
- Through `shell.env`, preserves the lease identity and authoritative session id for tool subprocesses without importing
  the daemon's full environment.

Duplicate tool events are harmless because the runner endpoints are idempotent. Plugin callback failure cannot fail a
tool call or change the worker's answer; process liveness and harness-agnostic boundaries remain the correctness
signals. A runner whose plugin never loads still executes work correctly and merely goes quiet between tool calls.

## Turn completion

The exit of the per-turn `opencode run` process is the only authoritative end of a turn. Nothing else closes a
generation: `session.idle` is a conversational event on the plugin channel, it does not appear in the worker's stdout
stream at all, and a worker can be idle with its process still alive and still holding the worktree. Session end is
therefore reported from the adapter's own completion boundary when the process exits, exactly as it is for a harness
whose CLI ends the turn by ending.

This costs OpenCode nothing that Claude Code has. An idle root is at most an early hint that a turn is winding down; it
never advances a node, never releases an environment, and never records a generation closed.

## Output and usage

The adapter parses newline-delimited OpenCode events into one invocation result. It concatenates completed root
assistant text in order, excludes tool output and child text from the verdict body, and surfaces explicit session
errors. Verdict and assessment retain Blizzard's `<Choice>...</Choice>` protocol.

Usage sums the invocation's completed model steps and carries the provider-reported model, input, output, reasoning,
cache read, cache write, and cost into Blizzard's usage shape. Missing fields remain zero only where OpenCode's schema
defines absence as no tokens. Cost needs more care than absence: a subscription-authenticated OpenCode step reports
`cost` as a literal `0` rather than omitting it, so the binding treats a zero cost under subscription authentication as
unknown and never as free. Tokens stay authoritative; a dollar figure Blizzard did not receive is never estimated. A
completed invocation must not be counted twice where both an event and an exported message describe the same model step.

## Models, effort, permissions, and compaction

OpenCode native models use `provider/model`, and its `--variant` flag carries the provider's reasoning effort. The
binding resolves a capability tier through the runner's OpenCode tier mapping into that pair, under the contract the
[harness selection specification](./harness-selection.md) owns; a non-namespaced value is accepted only when it is a
valid configured OpenCode model. The intended proving mapping reaches ChatGPT 5.6 Luna at the `max` variant. Where a
provider's variant vocabulary does not line up with Blizzard's effort words, the tier mapping absorbs the difference by
naming the variant directly, so no adapter invents a translation.

OpenCode `--auto` approves requests not explicitly denied; it is not equivalent to Claude Code's bypass mode. Explicit
runner, project, or managed denies remain effective, and the capability report describes this distinction. OpenCode's
non-interactive denial of its native question tool does not block `blizzard runner ask`, which remains an ordinary
lease-authenticated command.

OpenCode's compaction reserve and automatic-compaction switch do not represent Claude Code's numeric compaction
threshold. Until a harness-neutral policy names semantics both can honor, the OpenCode adapter resolves
`compaction_window` as unsupported and omits it. External subscription usage is not an adapter concern at all; the
[hardening specification](./hardening.md) owns where subscriptions are declared and sampled.
