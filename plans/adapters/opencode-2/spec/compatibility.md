# OpenCode 2 compatibility specification

The 2.x proof is the 2.x half of the existing `blizzard:manual-opencode-compatibility` verification method. The runner
diagnostic `blizzard runner opencode compatibility` reads the binary's version first and dispatches to the proof for the
observed line; a version in no admitted line is reported and nothing is probed. The 1.x proof is unchanged. A version is
compatible only when every required row passes together; a result from documentation alone is not evidence.

## Isolation

The proof never touches the operator's own OpenCode. It runs one private `opencode serve --hostname 127.0.0.1 --port 0`
under isolated `HOME`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_STATE_HOME`, `XDG_CACHE_HOME`, and `TMPDIR`, against a
disposable git repository, and stops it on every exit path. It never runs `opencode run` without `--server`, because a
bare client starts or joins the per-user managed service on its fixed port (49374), and an isolated home that tries to
start a second managed service fails on that port. It accepts the binary, model, and variant explicitly, prints the
observed version and one result per row, and writes sanitized evidence to a caller-named directory.

## Rows

| Row                   | Evidence required                                                                                                                                                                                                        |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Private server        | `serve` binds an ephemeral loopback port, reports its URL and credential to its launcher, loads configuration from the launcher's `OPENCODE_CONFIG*` environment, and exits with its process group leaving no survivor   |
| Server credential     | The launcher can set, or reliably read, the credential the API's basic auth demands, without reading the managed service's state file                                                                                    |
| Caller-named session  | `POST /api/session` accepts a runner-minted `ses_`-prefixed id, a location, a model with variant, a permission rule list, and an environment, and every later read reports that id                                       |
| Prompt and completion | `POST /api/session/{id}/prompt` starts a turn with arbitrary text (newlines and quotes preserved verbatim); the event stream and `GET /api/session/{id}` agree on when the root went idle and on its `outcome`           |
| Resume across servers | A session minted by one server process is prompted, read, and completed by a second server process on the same store after the first has exited                                                                          |
| Concurrent servers    | Two servers on one store run turns in two sessions at once without corrupting either, blocking either, or leaking configuration, environment, permission, or plugin events between them                                  |
| Process ownership     | Every tool subprocess descends from the server process and stays in its process group; killing the group ends every tool, and a killed server's session reads afterwards as `interrupted` or `failed`, never `running`   |
| Interrupt             | `POST /api/session/{id}/interrupt` stops a turn mid-tool, terminates the tool's processes, and records `outcome: interrupted`                                                                                            |
| Judgement             | A prompt to a resumed session yields final root assistant text from which `<Choice>` and the assessment can be isolated without tool output or child text                                                                |
| Nudge                 | A prompt with `queue` delivery to an idle session starts a turn; to a running session it is held to the turn's end rather than steered in                                                                                |
| Usage and cost        | Session, message, and step records carry model, variant, input, output, reasoning, cache read, cache write, and cost; the proof records whether a subscription-authenticated cost is ever nonzero                        |
| Errors and limits     | Provider refusal, usage limit with reset time, overload, unknown model, and unknown variant each produce a distinguishable, fixture-captured error shape                                                                 |
| Permission rules      | Session rules deny a tool and a shell pattern under every autonomy; a denied tool is absent or errors without ending the turn; an `ask` raises a permission request the runner can reply to over the API                 |
| Environment           | Session environment reaches shell tools and plugin hooks; nothing from one session's environment is visible in another's                                                                                                 |
| Plugin                | A runner-owned 2.x plugin loaded through configuration observes tool completion and session idle for root and child, tells them apart by `parentID`, and a plugin that fails to load does not affect the turn            |
| Ambient discovery     | The proof enumerates every configuration, plugin, skill, and MCP source the server reads outside the injected configuration — including `~/.claude/skills` — and which of them the injected configuration can switch off |
| Child agents          | A `subagent` call creates a child session whose `parentID` is the root, whose id is linked from the spawning tool part, whose usage is separately readable, and whose settlement is observable before the root completes |
| Transcript read       | `GET /api/session/{id}/message` returns the session's messages live and after completion, in stable ascending order, with stable ids, through compaction, without disturbing the writer                                  |
| Compaction            | Manual `compact` and automatic compaction at a configured buffer each produce a `compaction` message distinguishable from retained history                                                                               |
| Model and variant     | `provider/model#variant` selection survives fresh mint, resume, and judgement, and the model listing names the variants each model accepts                                                                               |
| Takeover              | `opencode --standalone --session <id>` in the recorded workdir, with the runner store and the lease identity environment, enters the session under attended permissions and continues it                                 |
| Version               | `opencode --version` prints one normalizable semantic version, and nothing in the proof's environment upgrades the binary                                                                                                |

- **Supported** — production behavior may depend on the observed contract, and the row carries a regression fixture.
- **Degraded** — the binding returns the harness-neutral absence the seam already defines, and the operator-facing
  capability says what is unavailable.
- **Blocking** — the step that depends on the behavior does not ship for that version.

## Already observed

A spike against 2.1.4 settled several rows, which enter the proof as regression fixtures rather than open questions. The
API's basic auth uses user `opencode`; caller-named `ses_` ids are accepted; deny rules hold, including shell patterns
and under `--auto`; `ask` is auto-rejected when no client answers; children carry `parentID` and are listed by
`GET /api/session?parentID=<root>`; root tokens exclude child usage; `session.interrupt` stops a tool and records
`outcome: interrupted`; manual compaction produces a `compaction` message carrying `metadata.keepID`; message ids are
stable and time-sortable; the event stream carries a durable per-session `seq`; and a bogus variant fails with
`provider.no-route`.

The spike also found what the proof must still settle: whether two servers can share one store, how a private server's
credential is set, whether `OPENCODE_CONFIG_CONTENT` can load a plugin, whether automatic compaction fires at the
configured buffer, and whether any ambient source escapes the injected configuration.

## Corpus

The proof writes its fixtures to `contracts/opencode/<version>/` beside the 1.x corpora, with a manifest naming the
line, every row's result, and the declared degradations. Each 2.x line's admitted range owes at least one committed
corpus inside it, checked at capability probe exactly as the 1.x range is. Widening a 2.x range is a proof run and a
committed corpus; nothing else may move it.
