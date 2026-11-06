# Runners, workspaces, and subscriptions on a running host

The three things a host holds a set of — its runners, its workspaces, its subscriptions — become records of the host,
created and retired through its API and taken up by its next host tick. Their tables are
[runner-store.md](./runner-store.md) §Tables; the routes' shape and the CLI are [runner-api.md](./runner-api.md); how
the host tick reconciles them is [passes.md](./passes.md) §The host tick.

## Runners

A hosted runner is the host's while its newest `hosted_runner_facts` row is `added`. Its id is minted by the hub and
`immutable`; everything else about it is its entries.

### Adding a runner to a running host

`POST /api/host/runners`, body `{ "name": "...", "projects": [...], "settings": { ... } }`, superuser only:

1. The host asks the hub to add the runner with the host token, as `runner add` does today (`runner/cli/runners.py:81`),
   and takes back the runner's id and token.
2. In one transaction it writes the `hosted_runners` row, an `added` fact, the runner's `runner_identity` row, its
   sealed token ([runner-store.md](./runner-store.md) §Sealed tokens), the runner's `runner_settings` row at revision
   `1`, and an entry for `name`, `projects`, and each key `settings` states, validated as a write is
   ([runner-store.md](./runner-store.md) §A write). The change row's op is `create`. Nothing is written to `.env`, and
   no directory is created.
3. It answers the runner's view with `state: "starting"`. The next host tick builds the runner — its object graph
   through `composition.add_runner` (`runner/composition.py:645`), which today only a test calls — puts it in the
   registry the app routes `/runners/{runner_id}` through (`app.state.runners`, `runner/app.py:453`), starts its tick
   driver, its trace-sweep driver where tracing is on, and its spawner executor, and runs its `resume.on_startup()` and
   takeover recovery as a boot does. Its first tick registers it.

`settings` is validated before step 1, so a body the store would refuse never mints a runner at the hub.

Nothing about the runners beside it changes, and nothing is written to `blizzard-host.toml`: the store is where the
runner lives. An operator who wants the file to say so too exports ([runner-api.md](./runner-api.md) §Apply).

### Retiring a runner from a running host

`POST /api/host/runners/{runner_id}/retire`, body `{ "force": false }`, superuser only, retires the runner at the hub
with the host token. The hub's refusal while the runner holds a chunk, unless forced, is the host's answer
(`execution/retirement.md`). Once the hub has retired it, the host writes a `retired` fact, and its next tick drains the
runner alone: stops its drivers, waits out its running tick, runs its `resume.on_shutdown()` — which, under `force`,
stops its workers as a host shutdown does (`ShutdownDrain`, `runner/lifecycle/shutdown_drain.py:28`) — closes its
process (`close_runner`, `runner/composition.py:375`), removes it from the app's registry, and deletes its sealed token.
Its identity row and its settings rows stay, as a retired hub record's history stays.

A runner the hub retires some other way — from the hub's CLI, or with its host — is noticed as today, by the hub
refusing its token, and `RetiredRunnerWatch` (`runner/hub/identity_service.py:43`) now writes the same `retired` fact,
so the same tick drains it. Today the watch only deletes the identity row and leaves the runner's threads running.

### The offline verbs

`blizzard runner add` and `blizzard runner retire` reach a running host's API, as every operator verb does. Against a
host that is not running they write the store directly, holding the host's lock as `retire` already does
(`store_write_guard`), through the same transaction the API runs, and say which they did; the host's next start takes
the runner up at its first tick. Neither edits `blizzard-host.toml` or `.env` any longer. A runner retired at the hub
while its host was down is noticed at the host's first tick, when the hub refuses its token, and drained there.

### A crash in an add

An add crosses the hub and the store, and a crash between the hub minting the runner and the store's transaction leaves
the runner minted at the hub with nothing on the host: no row, and no token, which was never written anywhere else.
Nothing runs under it; the hub shows it as a runner that has never registered, and the operator retires it from the hub
and adds again. `architecture/crash-correctness/runner.md` records the window under `bzh:crash-exemptions-runner`, since
the hub mints the id the store's rows are keyed by and the two cannot share a transaction.

## Workspaces and subscriptions

Each is a record in the shape `bzh:configured-record` gives the hub's: an `immutable` key — a workspace's
`workspace_id`, a subscription's `slug` and `provider` — a `revision` every write increments, a change row in
`host_setting_changes` beside every write, sparse-merge edits, retirement as an appended fact, and no delete.

| Verb   | Route                                                               |
| ------ | ------------------------------------------------------------------- |
| create | `POST /api/host/workspaces`, `POST /api/host/subscriptions`         |
| list   | `GET` the same; `?retired=true` includes retired records            |
| show   | `GET /api/host/workspaces/{workspace_id}`, `…/subscriptions/{slug}` |
| edit   | `PATCH` the same, with `If-Match`                                   |
| retire | `POST …/{key}/retire`                                               |
| enable | `POST …/{key}/enable`                                               |

A retire of a retired record, or an enable of an enabled one, writes nothing; an edit of a retired record is legal and
leaves it retired.

**Nothing is gated on use.** A workspace bound by every runner on the host can be retired, repointed to another root, or
moved to another provider, and a subscription spent by every runner can be retired; no write asks whether the record is
in use. What a change does to the work running through the record is fixed by the lifetimes, and the operator owns the
disk:

- **Creating a workspace** makes it the host's from the next host tick; it serves work once a runner binds a project to
  it. **Creating a subscription** samples it and reports it from that tick.
- **Editing a workspace** — its provider, root, repositories, pool, limit, base branch, or prompt — gives it a new
  provider at the next host tick, and the next environment acquired is acquired through it. An environment a claim holds
  stays held; when the claim releases or resumes it, it does so through the workspace as it now stands. Blizzard does
  not reconcile the disk: moving what is on disk to match a repointed workspace is the operator's, outside blizzard.
- **Editing a subscription** — its name, credentials path, usage API base, or sample interval — rebuilds its sampler and
  renewer at the next host tick. A running session keeps the credentials it launched with.
- **Retiring a workspace** stops the host reporting it at the next host tick. Each runner binding a project to it keeps
  running its other bindings and takes no claim through that one ([passes.md](./passes.md) §A runner's tick), and the
  hub flags the binding as unreported, as it flags one today. A chunk already claimed through it finishes there.
- **Retiring a subscription** stops the host sampling and reporting it. Each runner still naming it runs as though it
  named no subscription: from its next tick it declares none at registration, no subscription's usage throttles it, and
  the hub offers it work as it offers a runner paid for some other way. Naming the subscription again, once it is
  enabled, or naming another, is the runner's own write. Its usage samples and usage-limit marks stay.
- **Enabling** either reverses the retirement at the next host tick.

A workspace's `prompt`, `prompt_file`, and `prompt_package` are settings of its record. The existing
`PUT /api/host/workspaces/{workspace_id}/prompt` and its `DELETE` (`runner/api/workspace_prompt.py:56`) stay, write the
record through the same write as every other edit, and take its superuser check ([runner-api.md](./runner-api.md) §Who
may write). `PUT` sets `prompt` and returns `prompt_file` and `prompt_package` to their defaults, since a prompt set
through the route has always won over the configured one; `DELETE` returns all three to their defaults, so the workspace
is primed with no standing prompt, and its `source` answers `default`. The `workspace_prompt` table they wrote is
deprecated, read only by the seed ([runner-store.md](./runner-store.md) §Seeding), and dropped by a follow-up.

A workspace's or subscription's create and edit are validated as the host's file is at boot — a provider the host cannot
build, a root that is not a path, a `max_environments` below one — and refused with each key's error.
