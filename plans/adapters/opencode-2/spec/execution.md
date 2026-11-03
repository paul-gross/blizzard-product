# OpenCode 2 execution specification

## Composition

The OpenCode 2 adapter is a new binding beside the OpenCode 1 adapter, not a branch inside it. It implements the same
harness seam, returns harness-neutral values, and keeps model and variant mapping, driver composition, permission
translation, usage translation, and transcript access inside itself. It shares no parsing code with OpenCode 1: the two
versions' shapes are fixture-proven separately and may drift separately. Harness selection, the choice of binding by
observed version, and retry policy remain outside both.

The adapter's client is a small typed client over the stable `/api/*` operations the proof admits. It does not depend on
OpenCode's JavaScript SDK, and it calls no operation the proof marked experimental for any behavior a turn needs to
finish.

## The operator's OpenCode service

OpenCode 2 runs one managed background service per user, and Blizzard works through it. The runner never starts a
private server, never isolates a store, and never injects configuration into OpenCode: every session the fleet runs is a
session on the operator's own service, in the operator's own store, under the operator's own authentication and
configuration. Everything Blizzard controls about a session it sets through the API on that session. Blizzard never
tracks, signals, or kills an OpenCode process; the API is the only way it starts, stops, or interrupts work.

A driver finds the service through the registration OpenCode writes for it, and starts it the way any OpenCode client
does when none is registered. A service that cannot be reached within a bound fails the turn as a spawn failure.

The operator's configuration — global and project agents, MCP servers, skills, plugins, providers, and automatic
compaction — applies to fleet sessions as it applies to the operator's own, as the OpenCode 1 binding's ambient
configuration already does. Where the runner needs something narrower, it says so in the session's own rules: a session
can deny an MCP server's tools as it denies any tool.

## The driver

Every OpenCode 2 spawn, resume, judgement, and nudge launches one `blizzard runner opencode drive` process, and that
process is the worker the seam records: its exit is the end of the turn, and its stdout, or the judgement's
`output_path`, carries the turn's result. The seam's contract that a worker's exit declares its turn done is unchanged;
nothing in the runner loop, judgement, or selftest learns that a service exists. The driver is Blizzard's own API
client, launched through the runner's ordinary process launcher and held behind its trampoline until the provisional
spawn is durable. It starts no OpenCode process of its own.

## Session lifecycle

**Mint.** The adapter generates the session id — `ses_` followed by a ULID, accepted by OpenCode though it sorts
differently from native ids, which nothing in Blizzard depends on — and returns an identified handle at spawn, the path
the seam already provides for a harness that names its session at launch. The driver creates the session after the
launcher has released it: `POST /api/session` with the id, the spawn's working directory as its `location`, the resolved
model reference `{providerID, id, variant}`, the session's permission rules, and metadata marking it as the runner's and
naming its lease.

**Environment.** Immediately after create, and at the start of every resumed invocation, the driver sets
`PUT /api/session/{id}/environment` to the full allowlisted worker environment plus `BLIZZARD_*` identity. The call
replaces the shell environment wholesale and is held only in the service's memory, so it carries everything a tool needs
and is re-applied by every driver that opens the session. Each child session gets the same environment as it appears,
because a child that falls back to the service's environment would see the operator's, not the lease's.

**Prompt.** The prompt is the request body of `POST /api/session/{id}/prompt`, never a command-line argument, so it
reaches the model verbatim. The fresh-mint prompt prefix is prepended exactly as for OpenCode 1.

**Resume.** A resumed invocation re-applies the session's permission rules through `PATCH /api/session/{id}`, re-applies
its environment, and prompts the recorded id. It never sends a model on resume; the session keeps its own.

**Completion.** A turn is complete when the root session is idle, no descendant is running, and the root's inbox
(`GET /api/session/{id}/inbox`) is empty, observed after the last descendant settled — a finished background subagent
posts a message that wakes its root, so a root idle before its children finish is not yet done. The driver subscribes to
`GET /api/event` and keeps the events for its own session tree as a trigger only: the stream is volatile by contract, so
the driver also polls the session, its children, and its inbox on a bounded interval, and the session records are the
truth. The turn's outcome (`succeeded`, `failed`, `interrupted`), usage, cost, and final assistant message are read from
`GET /api/session/{id}` and its messages. The client-side `run --format json` stream is not used, because it routinely
ends before the final step is reported.

**Result.** The driver writes the turn's result to its stdout, or to the judgement's `output_path`, as Blizzard-owned
NDJSON (`opencode-v2-driver/1`): the session id, the outcome, the final root assistant text, per-step usage for the root
and each descendant, every permission and form decision it made, and any classified error. The adapter's verdict, usage,
and limit parsing read that record, never OpenCode's own output.

**Judgement.** Judgement is a resumed invocation whose final root assistant text yields `<Choice>` and the assessment.
Tool output, reasoning, and child-session text are never part of it.

**Nudge and answers.** A nudge or an answer delivery is a resumed invocation whose prompt uses `queue` delivery on an
idle session; the seam resumes only sessions whose worker has exited, so there is no running session to deliver into.
`steer` delivery is recorded as an available capability for `epic:steering` to adopt, and not used here.

**Children.** Subagent sessions run on the same service and count toward the turn. When the turn is asked to stop, the
driver interrupts every running descendant before the root.

## Heartbeat and lifecycle

The driver is the heartbeat. It runs `blizzard runner heartbeat` whenever it observes a tool complete in the root or a
descendant, and it reads the root's lifecycle from the session's own state, telling root from child by `parentID`. No
plugin is installed. Observing the root's lifecycle through the API lifts the 1.x root-hook degradation for OpenCode 2.

## Stopping a turn

The runner already stops workers with signals: SIGINT for shutdown drain and pause-park, SIGTERM then SIGKILL for
restart, cancellation, and forced takeover. The driver treats SIGINT and SIGTERM alike: it interrupts each running
descendant and then the root through `POST /api/session/{id}/interrupt`, waits a bounded grace for them to settle,
writes its result with outcome `interrupted`, and exits. The interrupt is the stop: OpenCode ends the tools the turn
started.

A driver that dies without writing its result — a SIGKILL, a crash — leaves its turn running on the service, because
nothing Blizzard kills reaches OpenCode. So the runner owns the last word: whenever a driver exits without a result, the
runner interrupts that session's tree through the API itself, and at every daemon start it lists the sessions its
metadata marks as its own and interrupts any still running. A turn that outlives its driver is stopped, never adopted.

A service that restarts or goes away mid-turn ends the turn as failed. The session keeps its history in the store, and
the next driver prompts it normally once the service is back.

## Permissions

Blizzard's fixed denies — the native question and form surfaces, and anything the runner forbids outright — are part of
each session's own rule list. A session's rules do not reach the subagents it spawns, which take their agent's
configured permissions from the operator's configuration; the driver applies the session's rule list to each child
through `PATCH /api/session/{id}` as the child appears, which covers a child from that moment, not from its first tool
call. That gap is accepted and declared, and closes when OpenCode lets a session's rules reach its children.

A session's ordered `permissions` list is composed from the autonomy level and ranks after agent configuration:

| Autonomy    | Session rules                                                                                     |
| ----------- | ------------------------------------------------------------------------------------------------- |
| `normal`    | The operator's rules, the fixed denies, then a final rule turning every remaining `ask` into deny |
| `auto`      | The operator's rules and the fixed denies; any remaining `ask` is approved by the driver          |
| `dangerous` | The operator's denies and the fixed denies; everything else allowed                               |

Over the API a permission request waits until something replies, so the driver replies to every one in its session tree
through `POST /api/session/{id}/permission/{requestID}/reply`: `reject` under `normal`, `once` under `auto` and
`dangerous` unless the request matches a deny. A form request (`session.form.*`) is cancelled under every autonomy. Each
reply is recorded in the driver's result, so nothing waits on a person and every answer leaves a trace. The 1.x
binding's `debug config` and `agent list` resolution is not ported.

`blizzard runner ask` remains an ordinary shell command, reachable under every autonomy.

## Models, variants, and compaction

The resolved model and effort become a model reference `{providerID, id, variant}`. Effort maps to a variant through
`[opencode.effort.aliases]`, defaulting to the identity mapping for `low`, `medium`, `high`, and `max`; the driver
checks the variant against the model listing (`GET /api/model`) before creating the session and refuses an unknown one,
rather than discovering `provider.no-route` mid-turn. Tiers map through `[opencode.models.aliases]`. An empty model
leaves OpenCode's configured default in force. The CLI's `provider/model#variant` form appears only in the takeover
command.

Automatic compaction is the operator's service-wide setting, and a session cannot carry its own threshold. The runner's
compaction window is enforced by the runner instead: it already reads a session's context size, and when the window is
crossed it requests a compaction through `POST /api/session/{id}/compact`, which is also how rotation policy compacts at
a precise moment.

## Errors and limits

Provider refusal, usage limit, and overload are classified from the session's error records, using the fixture-captured
shapes, and written into the driver's result. A usage limit carries its reset time into the resilience path exactly as
the 1.x classification does; an overload is retryable; an unknown model or variant is a configuration error the operator
must repair.

## Takeover

Takeover composes `blizzard runner opencode attend <id>` in the recorded working directory. The runner's existing
takeover path first stops the lease's worker, which the driver turns into an interrupt. The attend command then replaces
the session's unattended rules with the attended set through `PATCH /api/session/{id}` and opens the session in the
operator's own OpenCode with `opencode --session <id>`. The next automated resume re-applies the unattended rules, so
attended permissions never leak back into unattended work.
