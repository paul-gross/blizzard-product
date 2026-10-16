# Frontend contract

Two Angular apps change: the runner panel (`web/projects/runner`) becomes the host's panel, and the hub's runners view
(`web/projects/hub/src/app/runners`) learns hosts. Placement follows blizzard-context's frontend structure — code lives
with the daemon that serves it, and a view both daemons serve is written once in `fleet` with the daemon injected
(`architecture/frontend-structure/placement.md`, `bzh:frontend-placement`). Paths are relative to `web/projects/`.

## The host panel

The `runner` project keeps its name and becomes the panel the host serves at `/`. Its routes
(`runner/src/app/shell/app.routes.ts`) gain a level:

| Route                                | Page                                                                                        |
| ------------------------------------ | ------------------------------------------------------------------------------------------- |
| `/`                                  | The host: its runners, its workspaces and their environments, its subscriptions             |
| `/runners/:runnerId/board`           | Today's board (`board/board-page`), scoped to one runner                                    |
| `/runners/:runnerId/board/chunk/:id` | Today's chunk detail, scoped to one runner                                                  |
| `/runners/:runnerId/events`          | Today's events page, scoped to one runner                                                   |
| `/runners/:runnerId/machine`         | The runner's own status: harnesses, bindings with their agents in use, brake, pause control |

Today's top-level `/board` and `/events` routes are removed rather than redirected, and a host with one runner shows it
in the same shape as a host with several.

- **Runner selection.** The active runner is the `:runnerId` route parameter, read once in the shell and provided to
  every runner-scoped page; nothing below the shell reads the URL for it. The nav (`shell/nav/app-nav`,
  `mobile-tab-bar`) gains a runner switcher listing the host's runners with each one's brake state, so a braked runner
  is visible from any page.
- **The host page.** A new `host/` folder holds it. Environments (`environments/env-list`) move there, grouped by
  workspace, each held environment naming the runner and project binding that holds it. Subscriptions
  (`status/app-subscriptions`) move there, each naming the runners that spend it. Each runner appears as a row with its
  name, id, brake, bindings, and agents in use, linking to its board.
- **Per-runner pages.** `board/`, `events/`, and `machine/` keep their components; their queries take the active runner
  id. `machine/app-pause-control` lifts or sets that runner's brake alone, as the CLI's `--runner` does.
- **Sign-in.** `core/identity` signs in to the host once, against the host's `/api/auth/*`; there is no per-runner
  session ([process.md](./process.md) §Sign-in).

## Talking to the host

The host serves one OpenAPI document covering its own routes at the root (`/api/host/*`, `/api/auth/*`, `/api/health`,
`/api/ready`, and the event stream — [process.md](./process.md) §Listeners and the panel) and every runner's routes
under `/runners/{runner_id}/api/*`, with `runner_id` a path parameter. `openapi-ts.runner.config.ts` generates the
panel's client from it, so a runner-scoped call names its runner explicitly and the regenerated client is held to the
document by `web:client-drift`. Query keys (`runner/src/app/core/query-keys.ts`) and mutation keys
(`core/mutation-keys.ts`) gain the runner id as their first segment for every runner-scoped read and write, so two
runners' boards never share a cache entry.

The host's one event stream carries the runner id on every runner-scoped event. The panel's SSE handling routes each
event's cache invalidation by that id, and host-scoped events — a subscription sample, an environment freed — invalidate
the host page's reads.

The dev-server proxy (`web/proxy.conf.runner.js`) forwards both `/api` and `/runners` to the host's port, still read
from `BZ_RUNNER_PORT`.

## The hub's runners view

`hub/src/app/runners/runners-view` groups runners under their host. A new host card holds the host's name, last report,
workspaces with their environments in use, and subscriptions — `subscription-pace-group` moves from each runner row to
the host card, naming the runners that spend each plan — and lists its runners as today's runner rows (`runner-rows`). A
runner row under a host shows its bindings (`served-project-group` grows to show each project's workspace and agents)
and its one subscription's name rather than a roster. A runner with no host keeps today's row, under a heading of its
own. The host card carries the host's own verbs — retire, revoke and issue its credential — and each runner keeps its
pause, resume, and retire controls unchanged, with one more under a host: releasing it to stand alone
([hub.md](./hub.md) §Operator verbs).

## Shared between the two

The host's workspace-and-environment table and its subscription list are rendered by both apps, from different daemons.
Each is written once in `fleet` (`fleet/src/lib/`) and takes the daemon's client as an injected seam, per
`bzh:frontend-placement`. Everything else above is app-local.
