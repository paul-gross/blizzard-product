# OpenCode 2 execution specification

## Composition

The OpenCode 2 adapter is a new binding beside the OpenCode 1 adapter, not a branch inside it. It implements the same
harness seam, returns harness-neutral values, and keeps model and variant mapping, driver composition, permission
translation, plugin generation, usage translation, and transcript access inside itself. It shares no parsing code with
OpenCode 1: the two versions' shapes are fixture-proven separately and may drift separately. Harness selection, the
choice of binding by observed version, and retry policy remain outside both.

The adapter's server client is a small typed client over the stable `/api/*` operations the proof admits. It does not
depend on OpenCode's JavaScript SDK, and it calls no operation the proof marked experimental for any behavior a turn
needs to finish.

## The driver

Every OpenCode 2 spawn, resume, judgement, and nudge launches one `blizzard runner opencode drive` process, and that
process is the worker the seam records: its PID and start time are the lease's process, its exit is the end of the turn,
and its stdout, or the judgement's `output_path`, carries the turn's result. The seam's contract that a worker's exit
declares its turn done is unchanged; nothing in the runner loop, judgement, or selftest learns that a server exists.

The driver is launched through the runner's ordinary process launcher, so it is held behind the launcher's trampoline
until the provisional spawn is durable. Only then does it start OpenCode:

    opencode serve --stdio --port 0

as its own child, in its own process group, with the parent-death signal armed so the server cannot outlive it. The
driver generates a fresh credential for each server and passes it as `OPENCODE_PASSWORD`; the server announces its
loopback URL on stdout and removes the password from the environment its tools inherit. The server exits when the driver
closes its stdin. A server that does not announce and answer a readiness probe within a bound is killed, and the turn
fails as a spawn failure.

The server's environment is the driver's: the allowlisted worker environment, the `BLIZZARD_*` identity variables, the
OpenCode 2 store's `XDG_DATA_HOME`, `XDG_STATE_HOME`, and `XDG_CACHE_HOME`, `OPENCODE_CONFIG_CONTENT` carrying the
effective worker configuration, and `OPENCODE_DISABLE_AUTOUPDATE=1`. `OPENCODE_CONFIG_DIR` is never set, because it
replaces the operator's global configuration directory rather than adding to it; the operator bundle is merged into the
injected configuration, with its plugins referenced by absolute path. Because the server serves one turn and inherits
the driver's environment, a child session that falls back to the server's environment still sees the lease's own.

The driver never runs an OpenCode client command without `--server` or `--standalone`, so the operator's managed service
is never started or joined.

## Session lifecycle

**Mint.** The adapter generates the session id — `ses_` followed by a ULID, accepted by OpenCode though it sorts
differently from native ids, which nothing in Blizzard depends on — and returns an identified handle at spawn, the path
the seam already provides for a harness that names its session at launch. The driver creates the session after the
launcher has released it: `POST /api/session` with the id, the spawn's working directory as its `location`, the resolved
model reference `{providerID, id, variant}`, and the session's permission rules.

**Environment.** Immediately after create, and at the start of every resumed invocation, the driver sets
`PUT /api/session/{id}/environment` to the full allowlisted worker environment plus `BLIZZARD_*` identity. The call
replaces the shell environment wholesale and is held only in the server's memory, so it carries everything a tool needs
and is re-applied by every driver that opens the session.

**Prompt.** The prompt is the request body of `POST /api/session/{id}/prompt`, never a command-line argument, so it
reaches the model verbatim. The fresh-mint prompt prefix is prepended exactly as for OpenCode 1.

**Resume.** A resumed invocation starts a new private server on the same store, re-applies the session's permission
rules through `PATCH /api/session/{id}`, re-applies its environment, and prompts the recorded id. It never sends a model
on resume; the session keeps its own.

**Completion.** A turn is complete when the root session is idle, no descendant is running, and the root's inbox
(`GET /api/session/{id}/inbox`) is empty, observed after the last descendant settled — a finished background subagent
posts a message that wakes its root, so a root idle before its children finish is not yet done. The driver subscribes to
`GET /api/event` for `session.execution.*` events as a trigger only: the stream is volatile by contract, so the driver
also polls the session, its children, and its inbox on a bounded interval, and the session records are the truth. The
turn's outcome (`succeeded`, `failed`, `interrupted`), usage, cost, and final assistant message are read from
`GET /api/session/{id}` and its messages. A server that exits before completion is observed ends the turn as failed. The
client-side `run --format json` stream is not used, because it routinely ends before the final step is reported.

**Result.** The driver writes the turn's result to its stdout, or to the judgement's `output_path`, as Blizzard-owned
NDJSON (`opencode-v2-driver/1`): the session id, the outcome, the final root assistant text, per-step usage for the root
and each descendant, every permission and form decision it made, and any classified error. The adapter's verdict, usage,
and limit parsing read that record, never OpenCode's own output.

**Judgement.** Judgement is a resumed invocation whose final root assistant text yields `<Choice>` and the assessment.
Tool output, reasoning, and child-session text are never part of it.

**Nudge and answers.** A nudge or an answer delivery is a resumed invocation whose prompt uses `queue` delivery on an
idle session; the seam resumes only sessions whose worker has exited, so there is no running session to deliver into.
`steer` delivery is recorded as an available capability for `epic:steering` to adopt, and not used here.

**Children.** Subagent sessions run inside the driver's server and count toward the turn. When the turn is asked to
stop, the driver interrupts every running descendant before the root.

## Stopping a turn

The runner already stops workers with signals: SIGINT for shutdown drain and pause-park, SIGTERM then SIGKILL for
restart, cancellation, and forced takeover. The driver treats SIGINT and SIGTERM alike: it interrupts each running
descendant and then the root through `POST /api/session/{id}/interrupt`, waits a bounded grace for them to settle,
writes its result with outcome `interrupted`, stops the server, and exits. A server that does not settle inside the
grace is killed. SIGKILL of the process group ends the driver, the server, and every tool together, and a SIGKILL of the
driver alone takes the server with it through the parent-death signal. The runner's own record is the truth about a
killed turn: a session left with a stale outcome by a server that died is prompted normally by the next driver.

## Permissions

Blizzard's fixed denies — the native question and form surfaces, and anything the runner forbids outright — live in two
places, because a session's rules do not reach the subagents it spawns. They are written into every agent's permissions
in the worker configuration, which covers a child from its first tool call, and they are also part of each session's own
rule list. The driver additionally applies the session's rule list to each child through `PATCH /api/session/{id}` as
the child appears.

A session's ordered `permissions` list is composed from the autonomy level and ranks after agent configuration:

| Autonomy    | Session rules                                                                                     |
| ----------- | ------------------------------------------------------------------------------------------------- |
| `normal`    | The operator's rules, the fixed denies, then a final rule turning every remaining `ask` into deny |
| `auto`      | The operator's rules and the fixed denies; any remaining `ask` is approved by the driver          |
| `dangerous` | The operator's denies and the fixed denies; everything else allowed                               |

Over the API a permission request waits until something replies, so the driver replies to every one through
`POST /api/session/{id}/permission/{requestID}/reply`: `reject` under `normal`, `once` under `auto` and `dangerous`
unless the request matches a deny. A form request (`session.form.*`) is cancelled under every autonomy. Each reply is
recorded in the driver's result, so nothing waits on a person and every answer leaves a trace. The 1.x binding's
`debug config` and `agent list` resolution is not ported.

`blizzard runner ask` remains an ordinary shell command, reachable under every autonomy.

## Runner-owned plugin

The OpenCode 2 heartbeat plugin is a generated directory plugin, default-exporting `{ id, setup(ctx) }` with no package
dependency, listed in the worker configuration's `plugins` by absolute path. It runs `blizzard runner heartbeat` after
every tool execution in the root and in its descendants, observes `session.execution.*` events for the root as its
lifecycle signal, and distinguishes root from child through the session's `parentID`. The lifecycle signal lifts the 1.x
root-hook degradation for OpenCode 2. The plugin filters by session id even though its server serves one turn, so a
future shared-server mode cannot leak another lease's events into it. A plugin that fails to load degrades heartbeat and
the lifecycle signal without affecting the turn.

## Models, variants, and compaction

The resolved model and effort become a model reference `{providerID, id, variant}`. Effort maps to a variant through
`[opencode.effort.aliases]`, defaulting to the identity mapping for `low`, `medium`, `high`, and `max`; the driver
checks the variant against the model listing (`GET /api/model`) before creating the session and refuses an unknown one,
rather than discovering `provider.no-route` mid-turn. Tiers map through `[opencode.models.aliases]`. An empty model
leaves OpenCode's configured default in force. The CLI's `provider/model#variant` form appears only in the takeover
command.

A compaction window maps to `compaction.auto`, `compaction.buffer`, and `compaction.keep.tokens` in the injected
configuration, together with the model's `limit.input`. Those keys are server-wide, which here means per turn, because
each turn has its own server. Rotation policy that wants a compaction at a precise moment calls
`POST /api/session/{id}/compact` instead of waiting for the threshold.

## Errors and limits

Provider refusal, usage limit, and overload are classified from the session's error records, using the fixture-captured
shapes, and written into the driver's result. A usage limit carries its reset time into the resilience path exactly as
the 1.x classification does; an overload is retryable; an unknown model or variant is a configuration error the operator
must repair.

## Takeover

Takeover composes `blizzard runner opencode attend <id>` in the recorded working directory. The runner's existing
takeover path first stops the lease's worker, which the driver turns into an interrupt. The attend command then starts a
private server on the OpenCode 2 store with the lease identity environment and `OPENCODE_DISABLE_AUTOUPDATE=1`, replaces
the session's unattended rules with the attended set through `PATCH /api/session/{id}`, and runs the TUI as
`opencode --server <url> --session <id>`. When the operator leaves, it stops the server. The next automated resume
re-applies the unattended rules, so attended permissions never leak back into unattended work.
