---
epic: adapters/opencode-2
refinement: pristine
status: in-progress
---

# Plan — `epic:adapters/opencode-2`

An operator who has moved to OpenCode 2 finds the fleet will not follow them there. The OpenCode binding admits only the
1.x releases it was proven against, so a runner holding OpenCode 2 reports the harness as version-incompatible and never
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

None of that can be patched around, and none of it needs to be. OpenCode 2 is built to be driven as a service, so the
binding drives it that way: it works on the operator's own OpenCode, through its API alone. The API lets Blizzard name a
session before it exists, set that session's permissions and environment, read its transcript in order, interrupt it,
compact it on demand, see its children by parent, and price every step of it. Much of what the 1.x binding had to infer
from a stream of stdout, the 2.x binding can ask for, and nothing it does depends on owning a process.

## What the operator gets

Both major versions are first-class, and that is the destination, not a migration window. A runner works whichever major
version its binary is: point it at OpenCode 2 and it is a 2.x worker, with the same configuration it had as a 1.x one. A
machine runs one major version, and moving it is the operator's sequence: stop the runner, upgrade OpenCode, start the
runner. The fleet's runner host keeps running 1.x throughout. This epic adds 2.x and retires nothing: 1.x stays
admitted, and no removal is planned beside it.

A graph author keeps writing `opencode` and gets the major version of whichever runner takes the work. The OpenCode 2
binding resumes only sessions it minted, so a runner moved from 1.x to 2.x loses its open 1.x conversations and carries
on in fresh ones; conversations are ephemeral, and nothing more is done to preserve them. A fleet's 2.x sessions are the
operator's own OpenCode sessions, listed beside theirs and run under their configuration, and takeover opens one there.

Each binding admits its whole major version as a range, OpenCode 2 `>=2.0,<3.0` as OpenCode 1 admits 1.x. The proof runs
against one real 2.x release, 2.1.4 to start, and its fixtures are what the parsers are tested against; a later 2.x
release is trusted until it breaks, and a break is fixed when it shows up rather than prevented by pinning. A binary
outside both ranges is visible and refused. A binary that changes under a running runner — OpenCode 2 ships its own
`upgrade` command — is caught at the next launch, and the runner probes it again before offering it.

And the 2.x worker is good where 1.x was merely workable:

- **Identity is Blizzard's.** The runner names the session before the first prompt, so there is no handshake to time out
  and no stdout to scrape for an id.
- **Permission requests are answered.** Blizzard's fixed denies hold in the root session, and every permission request
  in the session's tree is answered by the runner, in the record, rather than left to wait on a person who is asleep. A
  subagent takes its agent's permissions until the runner applies the session's rules to it, a gap accepted until
  OpenCode lets a session's rules reach its children.
- **Usage is read, not reconstructed.** Cost, tokens, and outcome come from the session records themselves, per step,
  for the root and for each child.
- **Stopping is an interrupt.** A turn is stopped through the API, so the transcript records an interruption instead of
  a severed stream, and no OpenCode process is ever signalled.
- **The lifecycle arrives.** The 1.x binding declared it could not see a root session's lifecycle; the 2.x binding reads
  it from the session itself, and tells a root from a child by its parent.

## Decisions

Each question the build would otherwise stop on is decided here, so the build starts from choices.

**One harness id, two bindings, chosen by version.** OpenCode 2 is still OpenCode to the people choosing it, so it keeps
the `opencode` harness id. Behind the id sit two separate implementations, each owning its admitted versions and its
fixture corpus, and the version of the runner's configured binary selects between them. Configuration stays identical
across the two — the operator moves a runner to 2.x by changing its binary and nothing else — and graphs, the hub, and
the [harness selection contract](../harness-selection.md) see one `opencode` capability with an observed version, as
they do today.

**The operator's OpenCode, through its API alone.** OpenCode 2 runs one managed service per user, and the binding works
on it rather than beside it: fleet sessions live in the operator's store, under the operator's authentication and
configuration, on the same service as the operator's own. The runner starts that service with `opencode service start`
when it is not running, carrying none of the worker environment, and otherwise never runs a private server, isolates a
store, injects configuration, or tracks, signals, or kills an OpenCode process; the API is the only way it stops or
interrupts work, and everything it controls about a session it sets on that session. What only a service-wide setting
could give — automatic compaction thresholds, plugins, MCP servers — is the operator's, and the driver enforces the
lease's compaction window by requesting compaction when the window is crossed.

**A Blizzard driver owns each turn.** The runner's seam treats a worker's exit as the end of its turn, and that seam
stays as it is. So the recorded worker for every OpenCode 2 spawn, resume, and judgement is a Blizzard driver process: a
thin API client that creates or resumes the session, drives it, answers its permission requests, runs the heartbeat from
what it observes, writes the turn's result where the runner already reads one, and exits. A signal asking the turn to
stop becomes an API interrupt. Because killing a driver no longer kills its turn, the worker lifecycle seam gains two
harness-neutral additions: wherever the runner kills a driver or one exits without a result, it also interrupts that
session through the API, best effort, and at every start it interrupts its own recorded sessions still running with no
worker, before any restart resume.

**OpenCode 1 is guarded against the migration.** OpenCode 2 migrates a 1.x store in place the first time it opens it,
and both major versions use the operator's personal store. An operator who upgrades OpenCode without stopping a 1.x
runner migrates its sessions out from under it, so the 1.x binding's health learns to recognize a migrated store and
refuses it by name.

**Required behavior never stands on `/api/experimental/*`.** OpenCode labels its whole HTTP API experimental, so the
line the binding draws is the operations under `/api/experimental/*` — wait-until-idle, event replay, instruction
entries, export — which may enrich the binding behind a proven capability, while nothing a turn needs to finish depends
on them. The event stream is a prompt to look, never the record; the session's own state is the record.

**A turn includes its children.** A turn ends when the root session is idle, every descendant has settled, and nothing
is waiting in the root's inbox — so a background subagent cannot outlive the turn that started it, and a child's
completion cannot wake a root after Blizzard has declared it done.

**Questions are denied, as in 1.x.** OpenCode's native questions and forms are refused in unattended work; the worker
asks through `blizzard runner ask` exactly as the 1.x worker does. Routing native questions into Blizzard's channel is
worth doing, but it belongs to the human-channel work, not to this binding.

**Built by hand, on a machine that runs OpenCode 2.** No fleet runner host carries OpenCode 2, and a machine runs one
major version, so only an operator workstation that has it can produce honest evidence about it or exercise what is
built against it. The whole epic is worked there, by hand. Live evidence is still needed twice: at the start, where the
live proof writes the fixture corpus, and at the end, where the release gate runs a real chunk against a real
OpenCode 2. Everything between is built against what the evidence left behind — the committed corpus and a mock OpenCode
2 in `blizzard-mock` shaped from it — so its tests stay hermetic and run anywhere, including on the fleet's OpenCode 1
host.

**A mock OpenCode 2 is part of the epic.** Blizzard tests every harness against a mock that runs the prompt as a
behavior script and fails on command, and OpenCode 2 gets its own beside OpenCode 1's: a mock service answering the API
subset the driver uses, held to the same corpus as the real binary, so a test can script a turn that edits, spawns a
child, asks permission, hits a usage limit, or loses its service, without a real OpenCode or a real token.

## Sequence

The epic lands in the four steps `epic:adapters/opencode` proved, each gated on the one before.

1. **Prove OpenCode 2, by hand.** A 2.x compatibility proof runs against the operator's own OpenCode service, writes a
   fixture corpus for 2.1.4, and settles every row with evidence. The runner's compatibility diagnostic dispatches to
   the proof for the major version it observes, and an OpenCode 2 CLI-surface artifact in `blizzard-mock` lets the
   diagnostic's own tests run without OpenCode.
2. **A usable 2.x worker.** The mock OpenCode 2 lands first, shaped from the corpus. Against it, the driver spawns,
   resumes, judges, nudges, stops, and hands over a session, and the runner interrupts the session of every driver it
   kills.
3. **Conversation parity.** Transcripts, context measurement, usage, cost, and child agents are read from the service's
   API, normalized into Blizzard's vocabulary, and recognized by the hub's analytics under their own dialect. With the
   binding whole, the runner composes it when its binary is 2.x and drives it against the mock end to end.
4. **Hardening, closed by hand.** Crash verification of every stop path, reconciling orphaned turns at start, a version
   change under a running runner, health, the generic selftest, a hub from before this epic, and a runner moved across
   major versions are built against the mock; the release gate, with its soak of concurrent turns on one service, is the
   operator's run against a real OpenCode 2.

Every issue of the epic is filed for the operator, never ingested by the fleet, each blocked on the one before it in
landing order.

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
