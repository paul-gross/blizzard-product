# OpenCode 2 execution specification

## Composition

The line-2 adapter is a new binding beside the line-1 adapter, not a branch inside it. It implements the same harness
seam, returns harness-neutral values, and keeps model and variant mapping, server lifecycle, API translation, permission
translation, plugin generation, usage translation, and transcript access inside itself. It shares no parsing code with
line 1: the two lines' shapes are fixture-proven separately and may drift separately. Harness selection, line selection,
and retry policy remain outside both.

The adapter's server client is a small typed client over the stable `/api/*` operations the proof admits. It does not
depend on OpenCode's JavaScript SDK, and it does not call an operation the proof marked experimental for any behavior
the worker needs to finish a turn.

## The lease server

A line-2 lease runs one private server: `opencode serve --hostname 127.0.0.1 --port 0`, launched in the lease's first
environment workdir, in its own process group with the parent-death signal armed, carrying the lease's environment. That
environment is the allowlisted worker environment, the `BLIZZARD_*` identity variables, the line-2 store's
`XDG_DATA_HOME`, `XDG_STATE_HOME`, and `XDG_CACHE_HOME`, `OPENCODE_CONFIG_CONTENT` with the effective worker
configuration, and `OPENCODE_CONFIG_DIR` when an operator bundle exists. The server's PID and start time are the lease's
recorded process; its process group is what the runner kills.

The runner learns the server's URL and credential through the channel the proof admits, never by reading the managed
service's state file, and waits for a bounded readiness probe before issuing its first request. A server that does not
become ready inside the bound is killed and the launch fails as a spawn failure. The server lives exactly as long as the
lease: judgement, answer delivery, and nudges for the lease's session go through the same server, and the lease's end
stops it. A runner never sends a line-2 request to a server it did not launch, and never runs a line-2 `opencode`
command without `--server` or `--standalone`, so the operator's managed service is never started or joined.

## Session lifecycle

**Mint.** The runner generates the session id — `ses_` followed by a ULID — before launch and records it against the
lease in the same durable write that records the provisional process. Phase two of the two-phase spawn returns that id
immediately; there is no stdout handshake and no identity timeout. `POST /api/session` creates the session with the id,
the lease workdir as its location, the resolved model and variant, the session's permission rules, and the session
environment.

**Prompt.** The prompt is the request body of `POST /api/session/{id}/prompt`, never a command-line argument, so it
reaches the model verbatim. The fresh-mint prompt prefix is prepended exactly as for line 1.

**Resume.** A resumed lease launches a new server on the same store, re-applies the session's permission rules and
environment through the API — permissions and environment are re-asserted on every resume, never trusted from a previous
lease — and prompts the recorded id. It never sends a model on resume; the session keeps its own.

**Completion.** The runner subscribes to the server's event stream before prompting. A turn is complete when the root
session is idle and every descendant session has settled; the turn's result is then read from `GET /api/session/{id}`:
`outcome` (`succeeded`, `failed`, `interrupted`), usage, cost, and the final assistant message. The stream is the
trigger and the session record is the truth: a dropped stream falls back to polling the session record on a bounded
interval, and a server that exits without the root going idle completes the turn as failed. The client-side
`run --format json` stream is not used, because it routinely ends before the final step is reported.

**Judgement.** Judgement prompts the resumed session synchronously and reads the final root assistant text from the
session's messages after completion. `<Choice>` and the assessment are isolated from root text only; tool output,
reasoning, and child-session text are never part of it.

**Nudge and answers.** A nudge or an answer delivery is a prompt with `queue` delivery: to an idle session it starts a
turn, and to a running one it is held until the turn ends. The adapter does not use `steer` delivery in this epic; it
records `steer` as an available capability for `epic:steering` to adopt.

**Children.** Subagent child sessions run inside the lease server and count toward the turn. When the turn's root goes
idle with a descendant still running, the turn waits for it within the lease's ordinary bounds; a lease that ends for
any other reason interrupts every running descendant before stopping its server.

## Permissions

Each session carries an ordered `permissions` rule list, set at creation and re-asserted on resume, which is the last
word over operator and project configuration because it is the session's own. The runner composes it from the autonomy
level:

| Autonomy    | Session rules                                                                                                         |
| ----------- | --------------------------------------------------------------------------------------------------------------------- |
| `normal`    | The operator's rules, then a final rule turning every remaining `ask` into `deny`; the native question surface denied |
| `auto`      | The operator's rules with every `ask` answered by approval unless explicitly denied; question surface denied          |
| `dangerous` | Allow everything not explicitly denied; question surface denied                                                       |

Explicit denies from the operator bundle and from Blizzard's own fixed rules hold under every autonomy. A permission
request that reaches the API anyway is answered by the runner through `session.permission.reply` — rejected under
`normal`, approved under `auto` and `dangerous` unless it matches a deny — and recorded on the lease as a permission
event, so nothing ever waits on a person and every answer leaves a trace. The 1.x binding's `debug config` and
`agent list` resolution is not ported: the session's own rule list is authoritative, and the proof's permission row is
the evidence that it is.

`blizzard runner ask` remains an ordinary shell command, reachable under every autonomy.

## Runner-owned plugin

The line-2 heartbeat plugin is a generated directory plugin, default-exporting `{ id, setup(ctx) }` with no package
dependency, loaded through the worker configuration's `plugins` list. It runs `blizzard runner heartbeat` after every
tool execution in the root and in its descendants, observes root idle and root error as lifecycle signals, and
distinguishes root from child through the session's `parentID`. The lifecycle signal lifts the 1.x root-hook degradation
for line 2. Session environment set through the API supplies `BLIZZARD_*` identity to shell tools; the plugin does not
rewrite shell environments.

Because the server serves one lease, the plugin's event subscription sees only that lease's sessions. It still filters
by session id, so a future shared-server mode cannot leak another lease's events into it. A plugin that fails to load
degrades heartbeat and the lifecycle signal without affecting the turn.

## Models, variants, and compaction

The resolved model and effort become one reference, `provider/model#variant`. Effort maps to a variant through
`[opencode_2.effort.aliases]`, defaulting to the identity mapping for `low`, `medium`, `high`, and `max`; the adapter
checks the resolved variant against the model listing's `variants` before minting and refuses an unknown one, rather
than discovering `provider.no-route` mid-turn. Tiers map through `[opencode_2.models.aliases]`. An empty model leaves
OpenCode's configured default in force.

A compaction window maps to OpenCode's `compaction.buffer` and `compaction.keep` and the model's `limit.input` in the
worker configuration, applied per session. Rotation policy that wants a compaction at a precise moment calls
`POST /api/session/{id}/compact` instead of waiting for the threshold.

## Errors and limits

Provider refusal, usage limit, and overload are classified from the session's error records, not from text on stdout,
using the fixture-captured shapes. A usage limit carries its reset time into the resilience path exactly as the 1.x
classification does; an overload is retryable; an unknown model or variant is a configuration error the operator must
repair.

## Takeover

Forced or attended takeover interrupts the root session through the API, waits for it to settle, stops the lease server,
and then hands the operator `opencode --standalone --session <id>` in the recorded workdir, with the line-2 store, the
effective worker configuration, and the lease identity environment, under attended permissions. A standalone client owns
its private server, so the operator's session dies with their terminal and never lands in their own managed service.
