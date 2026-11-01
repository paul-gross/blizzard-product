# OpenCode 2 compatibility specification

The 2.x proof is the 2.x half of the existing `blizzard:manual-opencode-compatibility` verification method. The runner
diagnostic `blizzard runner opencode compatibility` reads the binary's version first and dispatches to the proof for the
observed line; a version in no admitted line is reported and nothing is probed. The 1.x proof is unchanged. A version is
compatible only when every required row passes together; a result from documentation alone is not evidence.

## Isolation

The proof runs the production driver against one private `opencode serve --stdio --port 0` per invocation, under an
isolated `XDG_DATA_HOME`, `XDG_STATE_HOME`, and `XDG_CACHE_HOME`, against a disposable git repository, and stops every
server on every exit path. Its other rows also isolate `HOME`, `XDG_CONFIG_HOME`, and `TMPDIR`, so a sentinel in the
operator's configuration cannot pass a row by accident; the ambient-discovery row deliberately does not, because it
exists to observe what production, which keeps both, actually reads. The proof never runs an OpenCode client command
without `--server` or `--standalone`, because a bare client starts or joins the per-user managed service on its fixed
port (49374).

The operator authenticates the proof's isolated store beforehand with the line-2 login command pointed at it. The proof
never prints, copies, or persists provider credentials, and its evidence is sanitized before it is written. It accepts
the binary, model, and variant explicitly, prints the observed version and one result per row, and writes evidence to a
caller-named directory.

## Rows

| Row                   | Evidence required                                                                                                                                                                                                                      |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Private server        | `serve --stdio --port 0` with `OPENCODE_PASSWORD` set announces its loopback URL, answers basic auth as user `opencode`, loads the injected configuration, keeps the password out of the tool environment, and exits when stdin closes |
| Startup cost          | Time to readiness and resident memory of a private server, recorded for the corpus at idle and with the worker configuration loaded; informational, not pass or fail                                                                   |
| Caller-named session  | `POST /api/session` accepts a runner-minted `ses_` id, a `location`, a model reference with variant, and a permission rule list, and every later read reports that id                                                                  |
| Session environment   | `PUT /api/session/{id}/environment` reaches shell tools and plugin hooks, replaces rather than merges, is absent in a new server until set again, and is invisible to every other session                                              |
| Prompt and completion | A prompt body with newlines and quotes reaches the model verbatim; root idle, settled descendants, and an empty inbox are observable by polling alone, with the event stream disconnected                                              |
| Resume across servers | A session minted by one server is prompted, read, and completed by a second server on the same store after the first has exited                                                                                                        |
| Concurrent servers    | Two servers on one store run turns in two sessions at once without corrupting either, blocking either, or leaking configuration, environment, permission, or plugin events between them                                                |
| Process ownership     | Every tool subprocess descends from the server and stays in its process group; killing the group ends every tool; a session whose server was killed mid-turn is prompted normally by the next server                                   |
| Interrupt             | `POST /api/session/{id}/interrupt` stops a turn mid-tool, terminates the tool's processes, and records `outcome: interrupted`; interrupting a running child before its root settles both                                               |
| Judgement             | A prompt to a resumed session yields final root assistant text from which `<Choice>` and the assessment can be isolated without tool output or child text                                                                              |
| Queue and steer       | A `queue` prompt to an idle session starts a turn; a `steer` prompt to a running session lands at its next step boundary                                                                                                               |
| Usage and cost        | Session, message, and step records carry model, variant, input, output, reasoning, cache read, cache write, and cost, for the root and each child separately; whether subscription-authenticated cost is ever nonzero is recorded      |
| Pricing               | `GET /api/model` carries per-model prices sufficient to estimate cost when a step reports none; otherwise the corpus records that estimates are unavailable                                                                            |
| Errors and limits     | Provider refusal, usage limit with reset time, overload, unknown model, and unknown variant each produce a distinguishable error shape                                                                                                 |
| Permission rules      | Session rules and agent configuration deny a tool and a shell pattern under every autonomy, in the root and inside a subagent; a denied call fails without ending the turn                                                             |
| Permission requests   | An `ask` raises a request that waits until `session.permission.reply` answers it, and a form request waits until it is cancelled; each reply takes effect                                                                              |
| Environment fallback  | A subagent with no session environment of its own sees the server's environment                                                                                                                                                        |
| Plugin                | A runner-owned plugin listed by absolute path in `OPENCODE_CONFIG_CONTENT` loads, observes tool completion and `session.execution.*` for root and child, tells them apart by `parentID`, and a load failure does not affect the turn   |
| Ambient discovery     | Under the operator's real `HOME` and `XDG_CONFIG_HOME`, every configuration, plugin, skill, and MCP source the server reads outside the injected configuration — including `~/.claude` and `~/.agents` — is enumerated                 |
| Read server           | A server whose configuration denies every tool serves session, message, and child reads without starting a turn                                                                                                                        |
| Child agents          | A `subagent` call creates a child whose `parentID` is the root and whose id is recorded on the spawning tool part; a finished background child wakes its root through the inbox, and that wake is observable                           |
| Transcript read       | `GET /api/session/{id}/message?order=asc` returns messages live and after completion with stable ids; an in-progress message read again by id reflects its completed state; tool states include `streaming`                            |
| Compaction            | Manual `compact` and automatic compaction at a configured `compaction.buffer` each produce a `compaction` message distinguishable from retained history                                                                                |
| Model and variant     | A model reference with variant survives fresh mint, resume, and judgement, and `GET /api/model` names the variants each model accepts                                                                                                  |
| Login                 | `opencode auth login --standalone` under the isolated store authenticates it without touching the managed service                                                                                                                      |
| Takeover              | A TUI attached with `--server <url> --session <id>` enters the session after its rules were replaced with the attended set, and continues it                                                                                           |
| Version               | `opencode --version` prints one normalizable semantic version, and nothing the driver or takeover launches upgrades the binary                                                                                                         |

- **Supported** — production behavior may depend on the observed contract, and the row carries a regression fixture.
- **Degraded** — the binding returns the harness-neutral absence the seam already defines, and the operator-facing
  capability says what is unavailable.
- **Blocking** — the step that depends on the behavior does not ship for that version.

## Contract settled for 2.1.4

Several rows have known answers for 2.1.4 and enter the proof as regression fixtures rather than open questions. The
API's basic auth uses user `opencode` and takes its password from `OPENCODE_PASSWORD`; `serve --stdio` announces its URL
and keeps the password from tools. Caller-named `ses_` ids are accepted. Deny rules hold, including shell patterns and
under `--auto`. A session's own rules do not reach its subagents, which take their agent's configured permissions.
Children carry `parentID` and are listed by `GET /api/session?parentID=<root>`; root usage excludes child usage.
`session.interrupt` stops a tool and records `outcome: interrupted`. A server that is not the managed service never
sweeps a session left mid-turn by a dead server. Manual compaction produces a `compaction` message. Message ids are
stable, and the default message order is descending. The event stream is volatile, and replay from a sequence number is
experimental. A bogus variant fails with `provider.no-route`. Autoupdate is disabled by `OPENCODE_DISABLE_AUTOUPDATE`
and not by project configuration, and `OPENCODE_CONFIG_DIR` replaces the global configuration directory.

What the proof must still settle is whether servers can share one store concurrently, whether a plugin loads through
`OPENCODE_CONFIG_CONTENT`, whether automatic compaction fires at the configured buffer, whether the session environment
reaches plugin hooks, whether a spawning tool part records its child's id, what the startup cost is, and whether any
ambient source escapes the injected configuration.

## Corpus

The proof writes its fixtures to `contracts/opencode-2/<version>/`, with a manifest naming the line, every row's result,
the startup measurement, and the declared degradations. Line 2 admits a version only when its corpus is committed.
Admitting another version is a proof run and a committed corpus; nothing else may move it.

## Companion changes

The proof extends `blizzard:manual-opencode-compatibility` rather than adding a method. The method's row in
`blizzard-context`'s verification matrix (`verification/blizzard.md`) pins the admitted range `>=1.18.25,<2.0`; the
change that lands the proof rewrites it to name both lines' admitted versions, and its manual-method detail gains the
2.x procedure. `blizzard:manual-opencode-compatibility-rehearsal` gains a 2.x walk against the line-2 CLI-surface
artifact, so the procedure is rehearsable offline.
