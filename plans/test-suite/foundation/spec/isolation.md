# Isolation

## The option

`isolation` is a Playwright option fixture, one of `common`, `fresh-project`, `fresh-tenant`, or `fresh-hub`, with no
default: a test in `tests/service/hub/` that never sets it fails before it starts, naming the option. A file or a
`describe` block sets it with `test.use({ isolation: 'fresh-tenant' })`, and the nearest setting wins, as for any
Playwright option. A test declares the loosest level its reads allow ([the plan](../index.md#isolation-levels) owns the
levels and why).

When `BLIZZARD_ISOLATION_RAISE=1`, the harness serves every test one level above the level it declared, and `fresh-hub`
stays `fresh-hub`. A test reads both through the `level` fixture, `level.declared` and `level.served`.
[The cross-check](#the-cross-check) is the flag's only intended user.

The option says where a test runs; it never says what the test needs there. That is the fixtures' job.

## The fixtures

A test names the fixtures it needs and builds none of them. Playwright makes each one only for a test that names it, or
that names a fixture resting on it, and hands every fixture in one test the same `tenant` and the same ledger, so
whatever a test names agrees on where it stands and what it owns.

### Where the test stands

| Fixture   | What it is                                                                             |
| --------- | -------------------------------------------------------------------------------------- |
| `level`   | `{ declared, served }`, the isolation level the test declared and the one it is served |
| `tenant`  | `{ id, name }`, the tenant at the served level                                         |
| `project` | `{ id, slug }`, the project at the served level                                        |

`tenant` and `project` resolve by the served level, and the level alone decides them; naming `project` never makes one
fresh:

| Level           | `tenant`                      | `project`                      | Built per test                                         |
| --------------- | ----------------------------- | ------------------------------ | ------------------------------------------------------ |
| `common`        | the hub's common tenant       | its `default` project          | nothing                                                |
| `fresh-project` | the hub's common tenant       | a project created for the test | `POST /api/projects`                                   |
| `fresh-tenant`  | a tenant created for the test | its `default` project          | `POST /api/admin/tenants`, through the internal client |
| `fresh-hub`     | the fresh hub's seeded tenant | its `default` project          | a [fresh hub](./hubs.md#fresh-hubs)                    |

A created tenant or project is named from the test's id and a random suffix, so two runs of the same test never collide.
Nothing a fixture builds is removed afterwards: the hub keeps it until the run stops the hub, as
[the plan](../index.md#the-shared-hub) decides.

### How the test acts and watches

Each client is bound to the test's tenant:

| Fixture       | What it is                                                                                                                                                                                                              |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api`         | The hub client, acting as the tenant's admin. Every request carries `X-Blizzard-Tenant: <tenant.id>`. `api.unfiltered` is the same client with its lists not narrowed ([Reads](#reads))                                 |
| `fleet(name)` | A fleet-plane client for one runner. It mints the runner with `POST /api/runners`, and sends its token as `Authorization: Bearer` on every `/api/fleet/` call. The token names the tenant, so it sends no tenant header |
| `forge`       | The hub's mock forge, reached at its own URL                                                                                                                                                                            |
| `events`      | The event stream ([below](#the-event-stream))                                                                                                                                                                           |
| `hubLog`      | The hub's log, filtered to `tenant` ([daemons.md](./daemons.md#logs))                                                                                                                                                   |

`api` always acts as the tenant's admin, the implicit operator on a hub with authentication off. Acting as anyone else
is explicit in the code, through a fixture that names that person, as the hub tier's `user` does.

### What the test makes

A precondition every test in a file shares is a fixture. What a test makes in numbers, or with arguments of its own, is
a factory fixture: a function the test calls, each call recorded in the ledger.

| Fixture                         | What it makes                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `issue(title, { repository? })` | An issue on the hub's forge, ingested with `POST /api/chunks`, both recorded as the test's own. Below `fresh-tenant` it files in the common tenant's shared `blizzard/common` ([hubs.md](./hubs.md#the-common-tenant)); from `fresh-tenant` up, in the repository it names, or in one `repository()` makes for it. Returns the issue and the chunk                                                                                           |
| `repository(name?)`             | At `fresh-tenant` and up, a repository on the hub's forge under owner `blizzard`, named from the test's id unless a name is given ([surfaces.md](./surfaces.md#repositories-on-demand)), then one `POST /api/config/apply` declaring the forge as the tenant's work source if it is not yet, the repository, and its link to `project`. Returns the repository's name and its work-source token prefix, so an ingest names `<name>:<number>` |
| `mockRunner(name, options)`     | At `fresh-tenant` and up, a mock runner against the test's hub with a token minted in `tenant`, stopped when the test ends. A mock runner left to add itself would name no tenant and be refused, so the harness never starts one without `--token`                                                                                                                                                                                          |

Secrets, when a test needs them, go through the secrets routes on `api`; nothing in the foundation's ports needs one.

## The clients and the ledger

The clients send what a test asks of them and keep two records for it. Nothing checks a call against the test's level
before it is sent: a test keeps to its level by the rules [rulebook.md](./rulebook.md#the-tiers-invariants) writes down,
and [the cross-check](#the-cross-check) is what notices when one does not. Only the factory fixtures that would
otherwise furnish the shared tenant check the level, since each is one condition: `fleet`, `repository`, and
`mockRunner` refuse below `fresh-tenant`, because a runner, a repository, or a work source is tenant-wide.

**The ledger** is the test's record of everything it created: each id a create call returns, under its kind. It lasts
for one test, and every fixture in the test shares it. The clients fill it from `harness/client/resources.ts`, a short
map of the resources tests create, each naming the create call that returns one and where its id sits in the answer, and
the list calls that return them and where their items sit:

```ts
chunk:   { creates: ['POST /api/chunks'], id: 'chunk_id', lists: { 'GET /api/chunks': 'items' } },
project: { creates: ['POST /api/projects'], id: 'project_id', lists: { 'GET /api/projects': 'items' } },
```

The map holds only the resources tests create, and grows when a port creates a new kind. A create call it does not know
is sent and answered as any other, and its result is simply not narrowed; nothing fails for want of an entry.

**A server error fails the test that caused it.** The clients read every answer, and a `5xx` fails the test that sent
the request the moment it arrives, naming the call and attaching the response body and the hub's log lines tagged with
the test's tenant around it. A server error is never an answer a black-box test asserts on by accident: a statement
guard's refusal surfaces this way ([daemons.md](./daemons.md#refusals-attributed)), and so does any other unhandled
fault. A test whose subject is a server error says so on the call, `api.expecting(503).get(…)`, and that one call's
`5xx` comes back to it instead.

## Reads

Whether a read is safe depends on what the test concludes from it, so that lives in the assertion, and the harness helps
in two ways.

At `common` and `fresh-project`, a list call `resources.ts` names comes back narrowed: the client keeps only the items
whose id the ledger holds under that resource's kind, so a test listing chunks sees the chunks it made. The whole list
takes `api.unfiltered`, a second client whose list calls are not narrowed, which a reviewer sees in the diff. At
`fresh-tenant` and `fresh-hub` the tenant is the test's own and lists come back whole.

What the client cannot narrow is the test's to keep, and [rulebook.md](./rulebook.md#the-tiers-invariants) writes it
down: at `common` and `fresh-project` a test names what it creates uniquely, asserts no count, emptiness, or absence
across the tenant, never finds anything by position, and chooses for a negative assertion's barrier an act only it would
emit. A log line with no `tenant_id` is a hub-wide read, and `hubLog.untagged()` refuses below `fresh-hub`.

## The event stream

`harness/events.ts` reads the hub's event stream with Node's own streaming `fetch`, because Playwright's request context
reads a response whole. It parses frames as `contracts/sse/` defines them, and keeps the last `id:` it saw, resuming
with `Last-Event-ID` when the connection drops. Event ids count per tenant from one.

`events.open()` subscribes before it returns, so a test opens it before the act it waits on. A frame can arrive before
the request that caused it returns. At `common` and `fresh-project` the reader passes on only frames that name an id the
ledger holds; `events.unfiltered` takes the whole stream. `waitFor(predicate, { timeout })` resolves with the first
matching frame, and its timeout is a failure, never a wait that ends quietly.

## The cross-check

A test placed too loose passes because of its neighbours, and fails the day it runs alone. The cross-check runs the
service projects twice over the same commit: once as declared, and once with `BLIZZARD_ISOLATION_RAISE=1`. Both runs use
`fullyParallel` at twice the CI worker count, so neighbours crowd each test far more than a pull request run does, and
both write the JSON report.

`harness/scripts/cross-check.ts <declared.json> <raised.json>` compares the two reports. It fails when a test passed in
one and not the other, naming each such test and the level it declared. A test that fails only as declared is placed too
loose. A test that fails only when raised is depending on sharing, a defect of its own.

Playwright runs files in a fixed order and has no shuffle, so the variety of neighbours comes from the parallelism, from
the two runs' different isolation, and from the weekly schedule running each time over a grown suite. The workflows that
run it are [ci.md's](./ci.md#the-cross-check-runs).
