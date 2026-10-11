# Host process contract

## Per host, per runner

`build_runner_process` (`runner/composition.py:226`) builds one `RunnerProcess` per runner, unchanged in what it holds
for that runner. A new `build_host_process` builds the host's own graph around them and owns what the runners share.

| Per host                                                                                  | Per runner                                                                                                          |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Listeners: one TCP port and one UDS socket, `<host dir>/host.sock`                        | Store and engine, `<dir>/data/runner.db`, with its identity row, outbox, brakes, and leases                         |
| The FastAPI app, its session secret, trusted proxies, `HubAuthModeCache`, JWKS cache      | `RunnerIdentityHolder`, `IdentityVerdictHolder`, `HubTokenHolder`, `VerifiedHubClient`, and its hub `httpx.Client`s |
| Panel sign-in: federation settings, cookie names, `RolePolicy`, `jwt_jti_seen` (§Sign-in) | `LoopContext`, `LoopConfig`, and one `PeriodicDriver` thread                                                        |
| The host store, `<host dir>/data/host.db`                                                 | The spawner executor, `HarnessRegistry`, `HarnessHealthCache`, `HarnessVersionCache`, published harness bundle      |
| One `IWorkspaceProvider` per workspace, with its lock ([workspaces.md](./workspaces.md))  | Platform-span identity stamp, lease-trace sweep, transcript pump                                                    |
| Subscription samplers and credential renewers ([subscriptions.md](./subscriptions.md))    |                                                                                                                     |
| `EventBroker`, `TelemetryReceiver`, span limiter, platform tracing provider               |                                                                                                                     |

Every runner keeps its own tick (`runner/loop/tick.py:56`) on its own thread, so one runner's slow step never delays
another's claim. The tick loses one step: `ExternalUsageSample` moves to the host
([subscriptions.md](./subscriptions.md)). The host keeps a tick of its own, on its own thread: it reports itself to the
hub (`POST /api/fleet/host`, [hub.md](./hub.md) §Registration), samples its subscriptions, and drains its own outbox. A
host whose credential the hub refuses keeps its runners working — their tokens are their own — and shows the refusal on
its panel.

## Isolation

A runner that cannot work stops alone. A refused token, an identity mismatch (`lifecycle/identity_check.py:138`), a
runner the hub does not hold under this host, a store that fails its migration, or a configuration error confined to one
`[[runner]]` entry halts that runner's tick and is shown on its view; the host and every other runner keep running. A
host-level error — an unreadable `blizzard-host.toml`, a port already bound, a host store that fails its migration —
stops the host, as a runner's equivalent stops it today. Which configuration errors are which is owned by
[configuration.md](./configuration.md) §Validation.

## One owner per runner directory

A machine may run several hosts — separate host directories, separate ports, separate runners — but a runner directory
is only ever driven by one process. On start, the host takes an exclusive, non-blocking advisory lock (`flock`) on
`<host dir>/data/host.lock`, and for each runner on `<dir>/data/runner.lock`, holding each for its lifetime; the kernel
releases them when the process dies, so a crash never leaves a stale lock. A host whose own lock is held stops. A runner
whose lock is held by another process is halted, its view naming the holder's pid. A runner daemon from before the host
takes no lock, so the host also probes `<dir>/runner.sock` and halts the runner when it answers. Keeping the
environments of two hosts on one machine apart is the operator's to declare, as keeping two runners' pools disjoint is
today.

## One tenant

The hub enforces it: a host is created in one tenant, every runner it adds is born there, and adoption refuses a runner
of another ([hub.md](./hub.md) §Operator verbs). The host store's `host_identity` row records the host id and tenant the
hub first answered with, and never changes after. Each runner's own tenant check is unchanged
([multi-tenancy runner slice](../../../delivered/multi-tenancy/runner.md)); a runner whose answered tenant differs from
the host's refuses to work through the gate an identity mismatch uses (`hub/verified.py:37`). The row is never edited: a
host moves to another tenant only by its operator retiring it, archiving its stores, and adding a new one. The one path
that sets it aside is a throwaway environment's re-add (§CLI, `--allow-readd`), which starts the host over rather than
moving it.

## Sign-in

Signing in to the panel is the operator, in a browser, proving who they are to the hub and coming back with a role. The
hub is the identity provider and the host its client: the host id is the `client_id` and the token's audience, and the
hub redirects back only to the callbacks the host reported ([hub.md](./hub.md) §Sign-in). The role is the person's role
in the host's tenant, mirrored through `[auth]` (`auth/roles.py:31`) as today. One sign-in covers the whole panel;
cookies are namespaced by host id (`auth/session.py:31`), and the federation flow (`api/federation.py:239`) is unchanged
but for whose identity it presents.

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

- `blizzard runner host [dir]` starts the host from `blizzard-host.toml`.
- `blizzard runner init [dir]` scaffolds a host. Given `--host-token`, it uses a host an operator already added; under
  an operator's `hub login` session it runs `hub host add` itself, in the session's tenant or `--tenant`, and writes the
  token to the host directory's `.env`. With `--allow-readd`, an init whose held host token the hub no longer knows —
  after a feature environment's hub data is reset — sets `host.db` and every runner store aside under timestamped names,
  as `runner store reset` does (`runner/runtime.py:161`), adds the host afresh, and re-adds each of its `[[runner]]`
  entries under the new host; a token the hub still knows is kept and nothing is set aside. Without the flag, an unknown
  token stops init. Feature environments, the mock fleet, and the test harnesses pass it, as they pass it to today's
  `init`; a production host never does.
- `blizzard runner add <name> [--project <p> --workspace <w> --agents <n>]…` asks the hub for a new runner under the
  host's credential (`POST /api/fleet/host/runners`), creates the runner's directory, writes its token to that
  directory's `.env`, and appends its `[[runner]]` entry. No hub sign-in is needed; the hub still mints the id.
- `blizzard runner retire --runner <name|id> [--force]` retires a runner under the host's credential and removes its
  entry, leaving its directory in place.
- `blizzard runner store reset <dir> --runner <name|id>` sets aside one runner's store, and refuses while the host's
  socket answers.
