---
epic: adapters/opencode-2
refinement: scaffolded
status: horizon
---

# Plan — `epic:adapters/opencode-2`

An operator who has moved to OpenCode 2 finds the fleet will not follow them there. The OpenCode binding admits only the
1.x line it was proven against, so a runner holding OpenCode 2 reports the harness as version-incompatible and never
offers it to a session. That refusal is the binding doing its job. OpenCode 2 is a new build, not a point release: a
spike against 2.1.4 found that very little of what the 1.x binding stands on survived. This epic gives OpenCode 2 a
binding of its own, built to be the better of the two rather than a port that limps.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What changed underneath

OpenCode 1 was a program: a turn was a process, and when the process ended, the turn had ended. OpenCode 2 is a service.
By default every `opencode` command is a thin client of one shared background server per user, and that server owns the
sessions, the configuration, the permissions, and the tools. Kill the client and the turn carries on without it. The
per-launch configuration the 1.x binding injects through its environment is silently ignored by a server that was
started by someone else, for someone else. Transcripts moved from `opencode export` to an HTTP API; the variant moved
into the model reference; the plugin interface was rewritten from scratch; `task` became `subagent` and `bash` became
`shell`; and the JSON a headless run prints usually stops before the step that would have carried the turn's final
usage.

Read as a list of breakages, that is daunting. Read as a list of affordances, it is the better harness. The server lets
Blizzard name a session before it exists, set that session's permissions and environment directly, read its transcript
live and in order, interrupt it cleanly, compact it on demand, see its children by parent, and price every step of it.
Everything the 1.x binding had to infer from a stream of stdout, the 2.x binding can ask for.

## What the operator gets

A runner offers OpenCode 1 and OpenCode 2 at the same time, and that is the destination, not a migration window. A
runner holding either line is a first-class worker; a runner holding both advertises both; the fleet's runner host keeps
running 1.x throughout. This epic adds 2.x and retires nothing: 1.x stays admitted, and no removal is planned beside it.

A graph author keeps writing `opencode` and gets whichever line the runner has, preferring the newer. An author who
needs one line writes `opencode@2` or `opencode@1`, and a runner without that line does not take the chunk. A session
stays on the line that minted it for its whole life, because the two lines keep separate conversation stores and neither
can resume what the other wrote.

Nothing runs on a version Blizzard has not proven. The 2.x binding admits one minor line at a time, starting at 2.1,
because the server API is young and renamed fields as recently as 2.1.0. A binary outside the admitted range is visible
and refused, never driven on hope; a binary that changes under a running runner — OpenCode 2 ships its own `upgrade`
command — is caught at the next launch, refused, and re-proven before it is offered again. Widening the range is a
compatibility proof and a committed fixture corpus, not an edit to a constant.

And the 2.x worker is good where 1.x was merely workable:

- **Identity is Blizzard's.** The runner names the session before the first prompt, so there is no handshake to time out
  and no stdout to scrape for an id.
- **Permissions are exact.** Each session carries its own ordered rule list, set at creation, so an unattended worker
  neither waits for a person nor slips a deny. A permission request that would have waited is answered by the runner, in
  the record, rather than by a timeout.
- **Usage is honest.** Cost, tokens, and outcome come from the session itself, per step, and a child agent's spend is
  counted against the turn that spawned it rather than disappearing.
- **Stopping is clean.** A turn is interrupted through the API before its process group is killed, so the transcript
  records an interruption instead of a severed stream.
- **Transcripts are live.** The board reads the conversation as it happens, in a stable order, children included.
- **The lifecycle hook arrives.** The 1.x binding declared it could not see a root session's lifecycle; the 2.x plugin
  can, and distinguishes a root from a child by its parent.

## Decisions

The spike left a handful of questions open. Each is decided here, so the build starts from choices rather than from a
survey.

**One harness id, two bindings.** OpenCode 2 is still OpenCode to the people choosing it, so it keeps the `opencode`
harness id. Behind the id sit two separate implementations, each owning its admitted range and its fixture corpus, and
the runner chooses between them by the version it observes, once, at capability probe. Authors select a line with an `@`
suffix on the acceptable-set entry; the [harness selection contract](../harness-selection.md) gains that suffix as an
additive extension.

**The worker is a private server per lease.** The lease's process is a runner-launched `opencode serve`, bound to
loopback, carrying the lease's environment and configuration, and serving that lease alone. The runner drives it over
the HTTP API. This keeps the invariant the 1.x specification insisted on — one lease, one process group, whose death
ends every tool it started — while gaining everything the server offers. The user's own background service is never
touched, never started, and never used.

**The runner owns the store.** A runner's 2.x sessions live in a runner-owned OpenCode data directory, shared by its
lease servers so that a session minted in one lease is resumable in the next. It is never the operator's personal store,
and never the 1.x store — OpenCode 2 migrates a 1.x store in place on first start, and a runner that let that happen
would strand every 1.x session it holds. The operator authenticates the runner's store once, through a runner command
that wraps `opencode auth login`.

**Required behavior stands only on stable endpoints.** Endpoints OpenCode marks experimental — wait-until-idle,
instruction entries, export — may enrich the binding behind a proven capability, but nothing the worker needs to finish
a turn depends on them.

**A turn includes its children.** A turn ends when the root session is idle and every descendant has settled, so a
background subagent cannot outlive the turn that started it and keep editing a worktree no one is watching.

**Questions are denied, as in 1.x.** OpenCode's native question surface stays denied in unattended work; the worker asks
through `blizzard runner ask` exactly as the 1.x worker does. Routing native questions into Blizzard's channel is worth
doing, but it belongs to the human-channel work, not to this binding.

**Built by hand, outside the fleet.** No fleet runner host carries OpenCode 2, and every step of this epic needs a live
one to be honest. It is built on an operator workstation that has OpenCode 2, and its issues stay out of the fleet's
intake until a runner host does.

## Sequence

The epic lands in the four steps the OpenCode epic proved, each gated on the one before.

1. **Prove the 2.x line.** A 2.x compatibility proof runs against a private server in an isolated store, writes a
   fixture corpus for 2.1.4, and settles every row with evidence. The runner's compatibility diagnostic dispatches to
   the proof for the line it observes.
2. **A usable 2.x worker.** The 2.x binding spawns, resumes, judges, nudges, and hands over a session through its
   private server, and the runner holds both lines at once, advertising and selecting by version.
3. **Conversation parity.** Transcripts, context measurement, usage, cost, and child agents are read from the server
   API, normalized into Blizzard's vocabulary, and recognized by the hub's analytics under their own dialect.
4. **Hardening.** Interruption, upgrade under a running runner, concurrent leases sharing a store, health, the generic
   selftest, and a release gate the operator runs by hand against a real OpenCode 2.

## Worth exploring next

Some of what OpenCode 2 offers is more than parity, and belongs to other epics. The binding records each as a capability
it can expose, without building the feature that would use it:

- **Steer and queue.** A message delivered at the next step boundary, or after the turn, is the native shape of
  `epic:steering`'s word to a running chunk.
- **Fork and revert.** Branching a session at a message is what `initiative:lab` needs to run arms from one shared
  history; reverting a turn is an undo the operator has never had.
- **A live event stream with durable sequence numbers.** The vital signs `initiative:code-harness-window` wants to show,
  delivered rather than polled.
- **Switching model mid-session.** Escalating a struggling session to a stronger tier without losing its conversation.
- **Session diffs and pairing links.** A reviewer's view of what a session changed, and an operator watching a session
  from a browser.
- **ACP.** OpenCode speaks the Agent Client Protocol; a binding to ACP itself could make the next harness a
  configuration rather than an epic.
