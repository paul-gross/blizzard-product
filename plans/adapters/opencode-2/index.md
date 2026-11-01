---
epic: adapters/opencode-2
refinement: scaffolded
status: horizon
---

# Plan — `epic:adapters/opencode-2`

An operator who has moved to OpenCode 2 finds the fleet will not follow them there. The OpenCode binding admits only the
1.x line it was proven against, so a runner holding OpenCode 2 reports the harness as version-incompatible and never
offers it to a session. That refusal is the binding doing its job. OpenCode 2 is a new build, not a point release, and
very little of what the 1.x binding stands on survived the rewrite. This epic gives OpenCode 2 a binding of its own,
built to be the better of the two rather than a port that limps.

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

None of that can be patched around, but most of it is an improvement once the binding speaks to the server directly. The
server lets Blizzard name a session before it exists, set that session's permissions and environment, read its
transcript in order, interrupt it cleanly, compact it on demand, see its children by parent, and price every step of it.
Much of what the 1.x binding had to infer from a stream of stdout, the 2.x binding can ask for.

## What the operator gets

A runner offers OpenCode 1 and OpenCode 2 at the same time, and that is the destination, not a migration window. A
runner holding either line is a first-class worker; a runner holding both advertises both; the fleet's runner host keeps
running 1.x throughout. This epic adds 2.x and retires nothing: 1.x stays admitted, and no removal is planned beside it.

A graph author keeps writing `opencode` and gets whichever line the runner has, preferring the newer. An author who
needs one line writes `opencode@2` or `opencode@1`, and a runner without that line does not take the chunk. A session
stays on the line that minted it for its whole life, because the two lines keep separate conversation stores and neither
can resume what the other wrote.

Nothing runs on a version Blizzard has not proven. The 2.x binding admits exactly the patch versions it has proven,
beginning with 2.1.4, because the server API is young and renamed fields as recently as 2.1.0. A binary outside the
admitted set is visible and refused, never driven on hope; a binary that changes under a running runner — OpenCode 2
ships its own `upgrade` command — is caught at the next launch, refused, and re-proven before it is offered again.
Admitting a new version is a compatibility proof and a committed fixture corpus, not an edit to a constant.

And the 2.x worker is good where 1.x was merely workable:

- **Identity is Blizzard's.** The runner names the session before the first prompt, so there is no handshake to time out
  and no stdout to scrape for an id.
- **Permissions are exact.** Blizzard's fixed denies hold in the root session and in every subagent it spawns, and a
  permission request is answered by the runner, in the record, rather than left to wait on a person who is asleep.
- **Usage is read, not reconstructed.** Cost, tokens, and outcome come from the session records themselves, per step,
  for the root and for each child.
- **Stopping is clean.** A turn is interrupted through the API before its processes are killed, so the transcript
  records an interruption instead of a severed stream.
- **The lifecycle hook arrives.** The 1.x binding declared it could not see a root session's lifecycle; the 2.x plugin
  can, and distinguishes a root from a child by its parent.

## Decisions

Each question the build would otherwise stop on is decided here, so the build starts from choices.

**One harness id, two bindings.** OpenCode 2 is still OpenCode to the people choosing it, so it keeps the `opencode`
harness id. Behind the id sit two separate implementations, each owning its admitted versions and its fixture corpus.
The runner configures each line in its own section, naming the binary for it, and the observed version confirms the
binary belongs to that line; a binary from the other line is refused, not quietly promoted. Authors select a line with
an `@` suffix on the acceptable-set entry; the [harness selection contract](../harness-selection.md) gains that suffix
as an additive extension.

**A Blizzard driver owns each turn, and each turn owns a private server.** The runner's seam treats a worker's exit as
the end of its turn, and that seam stays as it is. So the recorded worker for every line-2 spawn, resume, and judgement
is a Blizzard driver process. The driver starts a private OpenCode server as its own child, on loopback, with the turn's
environment and configuration; drives the session over the API; answers permission requests; writes the turn's result
where the runner already reads one; stops the server; and exits. One turn, one process group, whose death ends every
tool it started — the invariant the 1.x specification insisted on — while every API affordance is in reach. A signal
asking the turn to stop becomes an API interrupt before it becomes a kill. The operator's own background service is
never started, joined, or touched.

**The runner owns the store.** A runner's 2.x sessions live in a runner-owned OpenCode data directory, which every
driver's server opens, so a session minted in one turn resumes in the next. It is never the operator's personal store,
and never a 1.x store: OpenCode 2 migrates a 1.x store in place on first start, stranding every 1.x session in it. The
same hazard runs the other way. The 1.x binding keeps its sessions in the operator's personal store, so an operator who
also runs OpenCode 2 by hand on a runner host would migrate the 1.x runner's sessions out from under it. The 1.x
binding's health learns to recognize a migrated store and refuses it by name. The operator authenticates the 2.x store
once, through a runner command that wraps `opencode auth login`.

**Required behavior stands only on stable endpoints.** Endpoints OpenCode marks experimental — wait-until-idle, event
replay, instruction entries, export — may enrich the binding behind a proven capability, but nothing a turn needs to
finish depends on them. The event stream is a prompt to look, never the record; the session's own state is the record.

**A turn includes its children.** A turn ends when the root session is idle, every descendant has settled, and nothing
is waiting in the root's inbox — so a background subagent cannot outlive the turn that started it, and a child's
completion cannot wake a root after Blizzard has declared it done.

**Questions are denied, as in 1.x.** OpenCode's native questions and forms are refused in unattended work; the worker
asks through `blizzard runner ask` exactly as the 1.x worker does. Routing native questions into Blizzard's channel is
worth doing, but it belongs to the human-channel work, not to this binding.

**Live evidence by hand, the build by the fleet.** No fleet runner host carries OpenCode 2, so only an operator
workstation that has it can produce honest evidence about it. That evidence is needed twice: at the start, where the
live proof writes the fixture corpus, and at the end, where the release gate runs a real chunk against a real OpenCode
2. Everything between is built against what the evidence left behind — the committed corpus and a mock OpenCode 2 in
`blizzard-mock` shaped from it — which is work the fleet can do. The proof and the gate are the operator's; the binding
is the fleet's, once the corpus it builds from has landed.

**A mock OpenCode 2 is part of the epic.** Blizzard tests every harness against a mock that runs the prompt as a
behavior script and fails on command, and line 2 gets its own beside line 1's: a mock server answering the API subset
the driver uses, held to the same corpus as the real binary, so a test can script a turn that edits, spawns a child,
asks permission, hits a usage limit, or loses its server, without a real OpenCode or a real token.

## Sequence

The epic lands in the four steps the OpenCode epic proved, each gated on the one before.

1. **Prove the 2.x line, by hand.** A 2.x compatibility proof runs against a private server in an isolated store, writes
   a fixture corpus for 2.1.4, and settles every row with evidence. The runner's compatibility diagnostic dispatches to
   the proof for the line it observes, and a line-2 CLI-surface artifact in `blizzard-mock` lets the diagnostic's own
   tests run without OpenCode.
2. **A usable 2.x worker.** The mock OpenCode 2 lands first, shaped from the corpus. Against it, the driver spawns,
   resumes, judges, nudges, and hands over a session, and the runner holds both lines at once, advertising and selecting
   by line.
3. **Conversation parity.** Transcripts, context measurement, usage, cost, and child agents are read from the server
   API, normalized into Blizzard's vocabulary, and recognized by the hub's analytics under their own dialect.
4. **Hardening, closed by hand.** Interruption, upgrade under a running runner, concurrent turns sharing a store,
   health, and the generic selftest are built against the mock; the release gate is the operator's run against a real
   OpenCode 2, with line 1's live proof re-run beside it.

The fleet works steps 2 and 3 and the mock-tested half of step 4. Their issues are filed blocked on the step 1 issue,
which the operator closes when the corpus lands; the release gate is the operator's issue, blocked on the rest.

## Worth exploring next

Some of what OpenCode 2 offers is more than parity, and belongs to other epics. The binding records each as a capability
it can expose, without building the feature that would use it:

- **Steer and queue.** A message delivered at the next step boundary, or after the turn, is the native shape of
  `epic:steering`'s word to a running chunk.
- **Fork and revert.** Branching a session at a message is what `initiative:lab` needs to run arms from one shared
  history; reverting a turn is an undo the operator has never had.
- **A live event stream.** The vital signs `initiative:code-harness-window` wants to show, delivered rather than polled,
  with replay from a sequence number once OpenCode stabilizes it.
- **Switching model mid-session.** Escalating a struggling session to a stronger tier without losing its conversation.
- **Session diffs and pairing links.** A reviewer's view of what a session changed, and an operator watching a session
  from a browser.
- **ACP.** OpenCode speaks the Agent Client Protocol; a binding to ACP itself could make the next harness a
  configuration rather than an epic.
