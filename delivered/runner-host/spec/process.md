# Host process contract

## Per host, per runner

`build_runner_process` (`runner/composition.py:226`) builds one `RunnerProcess` per runner, holding what is that
runner's and a `RunnerStores` bound to its id ([store.md](./store.md) §Scoped access). A new `build_host_process` builds
the host's own graph around them and owns what the runners share.

| Per host                                                                               | Per runner                                                                                                          |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| The store and its engine, `<host dir>/data/host.db`, and `HostStores`                  | `RunnerStores`, bound to the runner's id                                                                            |
| Listeners: one TCP port and one UDS socket, `<host dir>/host.sock`                     | `RunnerIdentityHolder`, `IdentityVerdictHolder`, `HubTokenHolder`, `VerifiedHubClient`, and its hub `httpx.Client`s |
| The FastAPI app, its session secret, trusted proxies, `HubAuthModeCache`, JWKS cache   | `LoopContext`, `LoopConfig`, and one `PeriodicDriver` thread                                                        |
| Panel sign-in: federation settings, cookie names, `RolePolicy` (§Sign-in)              | The spawner executor, `HarnessRegistry`, `HarnessHealthCache`, `HarnessVersionCache`, published harness bundle      |
| One `IWorkspaceProvider` per workspace ([workspaces.md](./workspaces.md))              | Platform-span identity stamp, lease-trace sweep, transcript pump                                                    |
| Subscription samplers and credential renewers ([subscriptions.md](./subscriptions.md)) | Scratch directories under `runners/<name>/`                                                                         |
| `EventBroker`, `TelemetryReceiver`, span limiter, platform tracing provider            |                                                                                                                     |

Every runner keeps its own tick (`runner/loop/tick.py:56`) on its own thread, so one runner's slow step never delays
another's claim. The tick loses one step: `ExternalUsageSample` moves to the host
([subscriptions.md](./subscriptions.md)). The host keeps a tick of its own, on its own thread: it samples its
subscriptions and, once registered, reports itself to the hub (`POST /api/fleet/host`, [hub.md](./hub.md) §Registration)
and drains its own outbox. A host whose credential the hub refuses keeps its runners working — their tokens are their
own — and shows the refusal on its panel; so does a host whose hub answers its report with a not-found, a hub from
before hosts, while its facts wait in its outbox ([carry-over.md](./carry-over.md) §Hub and runners on different
clocks).

One tracing provider serves the host. Each runner's thread, and each request under a runner's routes, stamps that
runner's id on the spans it opens, as a lone runner stamps its own today (`contracts/traces/`), so the hub reads every
runner's steps as that runner's.

## Isolation

A runner that cannot work stops alone. A refused token, an identity mismatch (`runner/lifecycle/identity_check.py:138`)
— a token that resolves to any runner but the entry's `id` — a runner the hub does not hold under this host, or a
configuration error confined to one `[[runner]]` entry halts that runner's tick and is shown on its view; the host and
every other runner keep running. A host-level error — an unreadable `blizzard-host.toml`, a port already bound, a host
store that fails its migration — stops the host, as a runner's equivalent stops it today. Which configuration errors are
which is owned by [configuration.md](./configuration.md) §Validation.

## One owner per host directory

A machine may run several hosts — separate host directories, separate ports, separate runners — but a host directory,
and so its store, is only ever driven by one process. On start, and before any migration or ingest touches the store,
the host takes an exclusive, non-blocking advisory lock (`flock`) on `<host dir>/data/host.lock` and holds it for its
lifetime; the kernel releases it when the process dies, so a crash never leaves a stale lock. A host whose lock is held
stops, naming the holder's pid. A runner daemon from before the host takes no lock, so the host also refuses to start
while `<host dir>/runner.sock` answers, and every offline verb that moves or rewrites the store — `migrate` adopting
`runner.db`, `store reset`, ingest — refuses on the same two signs. Keeping the environments of two hosts on one machine
apart is the operator's to declare, as keeping two runners' pools disjoint is today.

Holding the directory is also what lets the host serialise its writers within one process: no other process can open its
store while it runs, so an in-process lock is enough wherever two of its own threads must not interleave
([workspaces.md](./workspaces.md) §The shared pool).

## One tenant

The hub enforces it: a host is created in one tenant, every runner it adds is born there, and adoption refuses a runner
of another ([hub.md](./hub.md) §Operator verbs). Once the host is registered, the store's `host_identity` row records
the host id and tenant the hub first answered with, and never changes after; an unregistered host holds one runner,
whose own tenant is the host's. Each runner's own tenant check is unchanged
([`epic:multi-tenancy/runner`](../../multi-tenancy/runner.md)); a runner whose answered tenant differs from the host's
refuses to work through the gate an identity mismatch uses (`runner/hub/verified.py:37`). The row is never edited: a
host moves to another tenant only by its operator retiring it, archiving its store, and adding a new one. The one path
that sets it aside is a throwaway environment's re-add (§CLI, `--allow-readd`), which starts the host over rather than
moving it.

## Sign-in

An unregistered host signs the operator in through its one runner, exactly as a runner does today. A registered host is
the client itself. Signing in to the panel is the operator, in a browser, proving who they are to the hub and coming
back with a role. The hub is the identity provider and the host its client: the host id is the `client_id` and the
token's audience, and the hub redirects back only to the callbacks the host reported ([hub.md](./hub.md) §Sign-in). The
role is the person's role in the host's tenant, mirrored through `[auth]` (`runner/auth/roles.py:33`) as today. One
sign-in covers the whole panel; cookies are namespaced by host id (`runner/auth/session.py:33`), the replay cache
(`jwt_jti_seen`) is the host's ([store.md](./store.md)), and the federation flow (`runner/api/federation.py:239`) is
unchanged but for whose identity it presents.

## Listeners and the panel

The host serves one origin. The panel at `/` is the host's, with a view per runner. Every route a lone runner served
that belongs to one runner — its leases, board, events, asks, control verbs, worker hooks, OTLP receiver — mounts under
`/runners/{runner_id}/`, keyed by the hub-minted id. The root keeps only what belongs to the host, and keeps it where
every probe already looks:

- `/api/health` and `/api/ready`, which answer for the host — its version, and each runner's state — so a supervisor's
  health check, a redeploy's version proof, and the single-writer probe (`runner/cli/daemon.py:112`) read them
  unchanged;
- `/api/auth/*`, the panel's one sign-in (§Sign-in);
- the host's event stream, whose runner-scoped events carry their runner's id ([frontend.md](./frontend.md) §Talking to
  the host);
- `/api/host/*`, the host's own routes: its workspaces, environments, subscriptions, and the list of its runners.

Nothing else answers at the unprefixed paths a lone runner served.

- **Workers.** `BLIZZARD_RUNNER_URL` (`runner/harness/harness_shared.py:63`) becomes
  `http://<host>:<port>/runners/<id>`, so worker hooks and OTLP reach the runner whose lease token they carry. A worker
  spawned before the host started is terminated by the restart that brings the host up and respawned by resume with the
  new address.
- **UDS.** A peer on `host.sock` is implicitly the superuser, as on `runner.sock` today. Operator verbs that reach a
  runner (`RunnerDaemon.reach`, `runner/cli/daemon.py:46`) gain `--runner <name|id>`, required when the host holds more
  than one runner and optional when it holds one, so `blizzard runner start --runner r-claude` lifts one runner's brake.

## CLI

Every flag and positional argument the runner's verbs take today keeps its meaning, since supervisors, provisioning, and
redeploy passes call them by those shapes: `host [dir]` with its `--dir`, `--host`, and `--port`; `migrate --dir`;
`init [dir]` with `--allow-readd`; and `--dir` on every verb that reaches a running daemon.

- `blizzard runner host [dir]` starts the host from `blizzard-host.toml`, and refuses a directory holding only
  `blizzard-runner.toml`, naming graduation ([carry-over.md](./carry-over.md) §Graduation). With `--check` it runs every
  check of [configuration.md](./configuration.md) §Validation, verifies the store's revision, and resolves each runner's
  token to its `id`, then exits — non-zero and naming each error if any fails — having started nothing.
- `blizzard runner migrate` migrates the host's store, adopting `data/runner.db` as `data/host.db` when the directory
  has graduated and holds no host store yet. A directory that has not graduated is refused, and nothing in it changes.
- `blizzard runner init [dir]` scaffolds a host with one runner, writing the runner's `id` and its token as
  `BZ_HUB_TOKEN_<NAME>`. It reaches the hub with the first credential it holds. A host token in its environment as
  `BZ_HOST_TOKEN` — never on its command line — uses a host an operator already added. An operator's `hub login` session
  runs `hub host add` itself, in the session's tenant or `--tenant`. Against a hub that admits anonymous adds, as a
  feature environment's open hub does, it adds the host anonymously, as it adds a runner today. It writes a new host's
  token to the host directory's `.env`. With `--allow-readd`, an init whose held host token the hub no longer knows —
  after a feature environment's hub data is reset — moves `host.db` aside under a timestamped name, as
  `runner store reset` does today (`runner/runtime.py:161`), adds the host afresh, and re-adds each of its `[[runner]]`
  entries under the new host; a token the hub still knows is kept and nothing is set aside. The same flag sets aside a
  store a runner from before the host left behind — a `data/runner.db` whose `blizzard-runner.toml` a provisioning step
  has cleared — and starts a host in its place. Without the flag, an unknown token or a store from before the host stops
  init, the latter naming graduation. Feature environments, the mock fleet, and the test harnesses pass it, as they pass
  it to today's `init`; a production host never does.
- `blizzard runner add <name> [--project <p> --workspace <w> --agents <n>]…` asks the hub for a new runner under the
  host's credential (`POST /api/fleet/host/runners`), creates `runners/<name>/`, writes its token to the host's `.env`
  as `BZ_HUB_TOKEN_<NAME>`, and appends its `[[runner]]` entry with the id the hub minted. It needs a registered host
  and no hub sign-in. A running host takes the new runner up at its next start, and `add` says so.
- `blizzard runner retire --runner <name|id> [--force]` retires a runner under the host's credential and removes its
  entry. A running host halts the runner at once, its token refused; its rows stay in the store until
  `store reset
  --runner` extracts them while the host is stopped.
- `blizzard runner ingest <source dir> --runner <name|id> [--workspace-id <id>]` moves a runner from another store into
  this host's ([carry-over.md](./carry-over.md) §Ingest). It is a verb of its own, so `host` keeps its directory
  argument.
- `blizzard runner store reset [dir] --runner <name|id>` extracts one runner's rows to an archive store; without
  `--runner` it moves the whole `host.db` aside ([store.md](./store.md) §Resetting one runner). Both refuse while the
  host's lock is held.
- The verbs that read one runner — `status`, `traces status`, `external-usage probe`, and the others that reach a
  running daemon — take `--runner` as the operator verbs do. `external-usage probe` reads the host's declared
  subscriptions, since a runner declares none of its own.
