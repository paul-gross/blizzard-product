# Showing a cadence

Everything an operator reads about a cadence is worked out in the hub, in `hub/domain/cadence/`, and answered on the
cadence's views, so the API, the CLI, and the board say the same thing. The board computes nothing of its own here,
unlike the routine's *blocked* badge, which it derives in the browser today
(`web/projects/hub/src/app/garden/routines/gardening-routines-page.model.ts:12`).

## Needs attention

A cadence's view carries `attention`: a list of what stops its errands, empty when nothing does. Each entry names its
chore by key, its scope when it stops one errand, and a reason. The reasons mirror the refusals of a hand-started run
(`plan_run`, `hub/domain/garden/runs/run.py:138`), so that whatever a cadence would be refused at mint, it shows
beforehand:

| Reason             | Stops                 | When                                                                             |
| ------------------ | --------------------- | -------------------------------------------------------------------------------- |
| `routine_unknown`  | every errand of chore | no routine in the cadence's project carries the chore's routine name             |
| `routine_retired`  | every errand of chore | the routine is retired                                                           |
| `graph_unresolved` | every errand of chore | no enabled graph carries the routine's graph name                                |
| `scope_unknown`    | one errand            | a listed slug names no scope in the project                                      |
| `scope_retired`    | one errand            | a listed scope, or the routine's default scope for a chore with none, is retired |
| `scope_unlinked`   | one errand            | a listed scope is not among the scopes the routine sweeps                        |
| `no_scopes`        | the chore             | an `all` chore whose routine has no live scope left                              |
| `no_chores`        | the cadence           | the cadence keeps no chore                                                       |

`reason` is an open vocabulary (`bzh:open-response-vocabulary`), so a later reason is additive. A retired cadence shows
no attention: it was stopped on purpose, and its view says it is retired. An `all` chore never reports a single scope
that has left the round; a scope retired or unlinked there is simply no longer one of its errands, as the plan decides.
The cadence goes on minting every errand that can still run.

## Errands

`GET /api/cadences/{cadence_id}/errands` answers every errand of the cadence, chore by chore in the cadence's order and
scope by scope in slug order, each with:

- the chore's key and the scope's slug;
- `state`: `due` when the next pass would mint it, `running` while its last run is unfinished, `waiting` until it comes
  due, or `stopped` when the cadence cannot run it (with the attention that stops it);
- its last run, when it has one: the chunk's id, its status, and its `scheduled_at`, `started_at`, and `finished_at`
  ([due.md](./due.md) §The three anchors);
- `due_at`, the forecast of [due.md](./due.md) §What a read shows as next, null while `running` or `stopped`.

The cadence's own view carries a count of its errands in each state, so a list of cadences shows at a glance which are
working, waiting, or stuck. `state` is an open vocabulary.

## Runs

A run's views — `RunSummary` on `GET /api/runs` and `GET /api/runs/{chunk_id}` (`hub/api/garden_runs.py:158`) — gain an
optional `cadence`, naming the cadence's id, slug, and name and the chore's key, for a run a cadence minted, and none
for a run started by hand. A run whose chore has since been removed still names its cadence and its removed chore's id,
with no key.

## The CLI

A `cadence` group in `hub/cli/cadence.py`, registered in `hub/cli/__init__.py`, with the conventions of `routine`
(`hub/cli/routine.py`): `FleetCommand`, `project_option`, and a cadence resolved by slug within its project.

```text
blizzard hub cadence create SLUG --project P [--name TEXT] [--cron EXPR --zone ZONE]
                                             [--spacing 10d] [--anchor started]
blizzard hub cadence list [--project P] [--retired]
blizzard hub cadence show SLUG --project P          # its fields, chores, attention, and errand counts
blizzard hub cadence edit SLUG --project P [--name TEXT] [the create options] [--no-schedule] [--no-spacing]
blizzard hub cadence retire|enable SLUG --project P [--by WHO]
blizzard hub cadence errands SLUG --project P      # every errand, its state, last run, and due_at

blizzard hub cadence chore add SLUG KEY --routine R [--scope S]... [--all-scopes]
                                        [--mode full|delta] [--note TEXT] [--hold]
blizzard hub cadence chore edit SLUG KEY [the add options] [--promote] [--default-scope]
blizzard hub cadence chore remove SLUG KEY
```

`--hold` writes `promote: false`, and `--promote` writes it back to true. A chore verb is an edit of its cadence, with
the cadence's revision and change row. A whole cadence is also declared in the config document through
`blizzard hub config apply`, which is the way to keep many of them. `contracts/cli/hub.json` is regenerated
(`blizzard:cli-contract`).

## The board

The gardening tab (`web/projects/hub/src/app/garden/gardening.routes.ts`) gains a fourth noun beside scopes, routines,
and runs: **Cadences**, at `/p/{project}/gardening/cadences`, a list and a detail route keyed by the cadence's slug,
laid out as the routines' are (`bzh:frontend-placement`, `bzh:frontend-container-presentational`).

- **The list** shows each cadence's name and slug, its rhythm — the cron expression and zone, its next moment, and its
  spacing (*`0 20 * * *` America/Chicago, next Thu 20:00, at least ten days apart*), its chores' count, its errand
  counts by state, and an attention badge when `attention` is not empty.
- **The detail** shows the rhythm with the schedule's next five moments, the attention list, and each chore with its
  errands in a table: scope, state, last run linked to the run's detail, and `due_at`. Its actions — edit, retire,
  enable, and add, edit, or remove a chore — are mutations over the routes, and each settles on refresh
  (`bzh:frontend-mutation-settles-on-refresh`).
- **A routine's detail** gains a *Kept by* line naming every cadence and chore that run it, so that a routine the plan's
  garden once said had no next-run time now shows when it runs next.
- **A run's detail** names the cadence and chore that minted it, where one did.

The project-agnostic lens shows the doorway the gardening tab already shows. While the `cadences` flag is off, the tab
is shown, and its list carries a line saying that cadences mint nothing until the flag is on.

## Operator documents

`docs/deployment/cadences.md` is the operator's page: what a cadence, a chore, and an errand are; the record's fields
with an example; when an errand is due, with the three anchors; promoting; attention; the CLI; and the board. It is
routed from `docs/deployment.md` and linked from `docs/deployment/routines-and-scopes.md` §Runs. The neighbouring pages
gain what touches them: `config-documents.md` the `cadences` section and its order after routines; `observability.md`
the `cadence` row in the table of sweeps, now seven; `board.md` the tab.
