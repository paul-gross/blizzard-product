# OpenCode 2 compatibility specification

The 2.x proof is the 2.x half of the existing `blizzard:manual-opencode-compatibility` verification method. The runner
diagnostic `blizzard runner opencode compatibility` reads the binary's version first and dispatches to the proof for the
observed major version; a version neither admits is reported and nothing is probed. The 1.x proof is unchanged. A
version is compatible only when every required row passes together; a result from documentation alone is not evidence.

## Where it runs

The proof runs the production driver against the operator's own OpenCode service, because that is what production
drives, in a disposable git repository. Every session it creates carries the proof's metadata, and the proof interrupts
and removes its sessions on every exit path, so it leaves nothing running and nothing behind in the operator's session
list. It uses the operator's own authentication and never prints, copies, or persists a provider credential, and its
evidence is sanitized before it is written. It accepts the binary, model, and variant explicitly, prints the observed
version and one result per row, and writes evidence to a caller-named directory.

## Rows

| Row                   | Evidence required                                                                                                                                                                                                                                                                                                                                                                          |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Managed service       | `opencode service start` starts the operator's service when none runs, under only the environment it is given, and whether it detaches the service from its own process group is recorded; `GET /api/info` answers; a disabled managed service is detectable; and the driver, an API client that is not OpenCode's own, finds the service through its registration and authenticates to it |
| Caller-named session  | `POST /api/session` accepts a runner-minted `ses_` id, a `location`, a model reference with variant, a permission rule list, and metadata, and every later read reports that id and metadata                                                                                                                                                                                               |
| Session environment   | `PUT /api/session/{id}/environment` reaches shell tools, replaces rather than merges, and is invisible to every other session; whether a child inherits its root's environment is recorded                                                                                                                                                                                                 |
| Prompt and completion | A prompt body with newlines and quotes reaches the model verbatim; root idle, settled descendants, and an empty inbox are observable by polling alone, with the event stream disconnected                                                                                                                                                                                                  |
| Service restart       | A session survives a restart of the service and is prompted normally afterward; a turn running when the service stopped is reported as no longer running once it returns                                                                                                                                                                                                                   |
| Concurrent sessions   | Two fleet sessions and an operator session run on one service at once without leaking rules, environment, permission requests, or events between them                                                                                                                                                                                                                                      |
| Interrupt             | `POST /api/session/{id}/interrupt` stops a turn mid-tool, ends the tool's processes, and records `outcome: interrupted`; interrupting a running child before its root settles both                                                                                                                                                                                                         |
| Judgement             | A prompt to a resumed session yields final root assistant text from which `<Choice>` and the assessment can be isolated without tool output or child text                                                                                                                                                                                                                                  |
| Queue and steer       | A `queue` prompt to an idle session starts a turn; a `steer` prompt to a running session lands at its next step boundary                                                                                                                                                                                                                                                                   |
| Usage and cost        | Session, message, and step records carry model, variant, input, output, reasoning, cache read, cache write, and cost, for the root and each child separately; whether subscription-authenticated cost is ever nonzero is recorded                                                                                                                                                          |
| Pricing               | `GET /api/model` carries per-model prices sufficient to estimate cost when a step reports none; otherwise the corpus records that estimates are unavailable                                                                                                                                                                                                                                |
| Errors and limits     | Provider refusal, usage limit with reset time, overload, unknown model, and unknown variant each produce a distinguishable error shape                                                                                                                                                                                                                                                     |
| Permission rules      | Session rules deny a tool, a shell pattern, and an MCP server's tools under every autonomy in the root; a denied call fails without ending the turn; what a subagent may do before and after the driver applies the session's rules to it is recorded                                                                                                                                      |
| Permission requests   | An `ask` raises a request, in the root or a child, that waits until `session.permission.reply` answers it, and a form request waits until it is cancelled; each reply takes effect                                                                                                                                                                                                         |
| Events                | The event stream carries each event's session id, and enough to tell a root from its children, for a driver to keep only its own session tree's events                                                                                                                                                                                                                                     |
| Child agents          | A `subagent` call creates a child whose `parentID` is the root and whose id is recorded on the spawning tool part; a finished background child wakes its root through the inbox, and that wake is observable                                                                                                                                                                               |
| Transcript read       | `GET /api/session/{id}/message?order=asc` returns messages live and after completion with stable ids; an in-progress message read again by id reflects its completed state; tool states include `streaming`                                                                                                                                                                                |
| Compaction            | A requested `compact` produces a `compaction` message distinguishable from retained history                                                                                                                                                                                                                                                                                                |
| Model and variant     | A model reference with variant survives fresh mint, resume, and judgement, and `GET /api/model` names the variants each model accepts                                                                                                                                                                                                                                                      |
| Takeover              | `opencode --session <id>` opens the session in the operator's OpenCode after its rules were replaced with the attended set, and continues it                                                                                                                                                                                                                                               |
| Version               | `opencode --version` prints one normalizable semantic version, and `GET /api/info` reports the version the service runs                                                                                                                                                                                                                                                                    |
| 1.x store migration   | Against a disposable 1.x store, what OpenCode 2 changes on disk when it first opens it, whether OpenCode 1 can still read it afterward, and the signal OpenCode 1's health reads to recognize it, committed as a fixture beside the 1.x corpus                                                                                                                                             |

- **Supported** — production behavior may depend on the observed contract, and the row carries a regression fixture.
- **Degraded** — the binding returns the harness-neutral absence the seam already defines, and the operator-facing
  capability says what is unavailable.
- **Blocking** — the step that depends on the behavior does not ship for that version.

## Contract settled for 2.1.4

Several rows have known answers for 2.1.4 and enter the proof as regression fixtures rather than open questions.
Caller-named `ses_` ids are accepted. Deny rules hold, including shell patterns and under `--auto`. A session's own
rules do not reach its subagents, which take their agent's configured permissions. Children carry `parentID` and are
listed by `GET /api/session?parentID=<root>`; root usage excludes child usage. `session.interrupt` stops a tool and
records `outcome: interrupted`. Manual compaction produces a `compaction` message. Message ids are stable, and the
default message order is descending. The event stream is volatile, and replay from a sequence number is experimental. A
bogus variant fails with `provider.no-route`.

What the proof must still settle is how the managed service is found, started, and authenticated by a client that is not
OpenCode's own and whether `service start` detaches it, whether events carry the session id, whether a child inherits
its root's environment, whether a spawning tool part records its child's id, and what migrating a 1.x store actually
changes.

## Corpus

The proof writes its fixtures to `contracts/opencode/<version>/`, with a manifest naming the major version, every row's
result and the declared degradations. The corpus is what the parsers and the mock are held to; it does not gate
admission, which is the major version's range. The proof is rerun, and a new corpus committed, when a 2.x release breaks
the binding.

## Companion changes

The proof extends `blizzard:manual-opencode-compatibility` rather than adding a method. The method's row in
`blizzard-context`'s verification matrix (`verification/blizzard.md`) pins the admitted range `>=1.18.25,<2.0`; the
change that lands the proof rewrites it to name both major versions' admitted versions, and its manual-method detail
gains the 2.x procedure. `blizzard:manual-opencode-compatibility-rehearsal` gains a 2.x walk against the OpenCode 2
CLI-surface artifact, so the procedure is rehearsable offline.

The binding adds two runner commands, `blizzard runner opencode drive` and `blizzard runner opencode attend`, which
carry their `blizzard:cli-contract` proof and their change notes under `blizzard:change-notes`. The version-selected
binding ([versions](./versions.md#binding-dispatch)) carries its amendment to the runner's architecture guidance in
`blizzard-context`, landing with the dispatcher.
