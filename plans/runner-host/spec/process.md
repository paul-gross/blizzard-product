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
own — and shows the refusal on its panel.

## Isolation

A runner that cannot work stops alone. A refused token, an identity mismatch (`lifecycle/identity_check.py:138`), a
runner the hub does not hold under this host, or a configuration error confined to one `[[runner]]` entry halts that
runner's tick and is shown on its view; the host and every other runner keep running. A host-level error — an unreadable
`blizzard-host.toml`, a port already bound, a host store that fails its migration — stops the host, as a runner's
equivalent stops it today. Which configuration errors are which is owned by [configuration.md](./configuration.md)
§Validation.

## One owner per host directory

A machine may run several hosts — separate host directories, separate ports, separate runners — but a host directory,
and so its store, is only ever driven by one process. On start, and before any migration or ingest touches the store,
the host takes an exclusive, non-blocking advisory lock (`flock`) on `<host dir>/data/host.lock` and holds it for its
lifetime; the kernel releases it when the process dies, so a crash never leaves a stale lock. A host whose lock is held
stops, naming the holder's pid. A runner daemon from before the host takes no lock, so the host also refuses to start
while `<host dir>/runner.sock` answers. Keeping the environments of two hosts on one machine apart is the operator's to
declare, as keeping two runners' pools disjoint is today.

## One tenant

The hub enforces it: a host is created in one tenant, every runner it adds is born there, and adoption refuses a runner
of another ([hub.md](./hub.md) §Operator verbs). Once the host is registered, the store's `host_identity` row records
the host id and tenant the hub first answered with, and never changes after; an unregistered host holds one runner,
whose own tenant is the host's. Each runner's own tenant check is unchanged
([multi-tenancy runner slice](../../../delivered/multi-tenancy/runner.md)); a runner whose answered tenant differs from
the host's refuses to work through the gate an identity mismatch uses (`hub/verified.py:37`). The row is never edited: a
host moves to another tenant only by its operator retiring it, archiving its store, and adding a new one. The one path
that sets it aside is a throwaway environment's re-add (§CLI, `--allow-readd`), which starts the host over rather than
moving it.

## Sign-in

An unregistered host signs the operator in through its one runner, exactly as a runner does today. A registered host is
the client itself. Signing in to the panel is the operator, in a browser, proving who they are to the hub and coming
back with a role. The hub is the identity provider and the host its client: the host id is the `client_id` and the
token's audience, and the hub redirects back only to the callbacks the host reported ([hub.md](./hub.md) §Sign-in). The
role is the person's role in the host's tenant, mirrored through `[auth]` (`auth/roles.py:31`) as today. One sign-in
covers the whole panel; cookies are namespaced by host id (`auth/session.py:31`), the replay cache (`jwt_jti_seen`) is
the host's ([store.md](./store.md)), and the federation flow (`api/federation.py:239`) is unchanged but for whose
identity it presents.

## Listeners and the panel

The host serves one origin. The panel at `/` is the host's, with a view per runner. Each runner's API routes mount under
`/runners/<runner id>/`, keyed by the hub-minted id, and the host's own routes — its workspaces, environments,
subscriptions, and the list of its runners — sit at `/api/host/`. Nothing answers at the unprefixed paths a lone runner
served.

- **Workers.** `BLIZZARD_RUNNER_URL` (`harness/harness_shared.py:63`) becomes `http://<host>:<port>/runners/<id>`, so
  worker hooks and OTLP reach the runner whose lease token they carry. A worker spawned before the host started is
  terminated by the restart that brings the host up and respawned by resume with the new address.
- **UDS.** A peer on `host.sock` is implicitly the superuser, as on `runner.sock` today. Operator verbs that reach a
  runner (`cli/daemon.py:97`) gain `--runner <name|id>`, required when the host holds more than one runner, so
  `blizzard runner start --runner r-claude` lifts one runner's brake.

## CLI

- `blizzard runner host [dir]` starts the host from `blizzard-host.toml`, and refuses a directory holding only
  `blizzard-runner.toml`, naming graduation ([carry-over.md](./carry-over.md) §Graduation).
- `blizzard runner migrate [dir]` migrates the host's store, adopting `data/runner.db` as `data/host.db` when the
  directory has graduated and holds no host store yet.
- `blizzard runner init [dir]` scaffolds a host with one runner. Given `--host-token`, it uses a host an operator
  already added; under an operator's `hub login` session it runs `hub host add` itself, in the session's tenant or
  `--tenant`, and writes the token to the host directory's `.env`. With `--allow-readd`, an init whose held host token
  the hub no longer knows — after a feature environment's hub data is reset — moves `host.db` aside under a timestamped
  name, as `runner store reset` does today (`runner/runtime.py:161`), adds the host afresh, and re-adds each of its
  `[[runner]]` entries under the new host; a token the hub still knows is kept and nothing is set aside. Without the
  flag, an unknown token stops init. Feature environments, the mock fleet, and the test harnesses pass it, as they pass
  it to today's `init`; a production host never does.
- `blizzard runner add <name> [--project <p> --workspace <w> --agents <n>]…` asks the hub for a new runner under the
  host's credential (`POST /api/fleet/host/runners`), creates `runners/<name>/`, writes its token to the host's `.env`
  as `BZ_HUB_TOKEN_<NAME>`, and appends its `[[runner]]` entry. It needs a registered host and no hub sign-in; the hub
  still mints the id.
- `blizzard runner retire --runner <name|id> [--force]` retires a runner under the host's credential, extracts its rows
  to an archive store as a reset does, and removes its entry.
- `blizzard runner host ingest <source dir> --runner <name|id>` moves a runner from another store into this host's
  ([carry-over.md](./carry-over.md) §Ingest).
- `blizzard runner store reset [dir] --runner <name|id>` extracts one runner's rows to an archive store; without
  `--runner` it moves the whole `host.db` aside ([store.md](./store.md) §Resetting one runner). Both refuse while the
  host's lock is held.
