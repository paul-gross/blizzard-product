# The hub's settings

Where the hub keeps its `pass` settings, who may change them, the routes, the document an apply takes, and how each
long-lived object built from them today reads them per pass instead. Which keys move is [settings.md](./settings.md)
§Hub.

## One record for the hub, not for a tenant

Every configured record the hub has today is a tenant's: its table carries `tenant_id`, its change rows go to
`config_changes` (`hub/store/schema.py:703`, `tenant_id` not null), its revisions are read through `ScopedConnections`,
its cache is the tenant's `LiveConfig` (`hub/composition.py:849`), and its routes are tenant-bound. The settings this
epic moves are the hub's, shared by every tenant it serves — one transcript cap, one trace export, one login throttle —
so they are kept in one hub-wide record beside the tenant records, never in them.

| Table                 | Holds                                                                                                                               |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `hub_settings`        | One row: `revision`, `updated_at`                                                                                                   |
| `hub_setting_entries` | `key` (PK), `value` (JSON, nullable), `at_default`, `revision`                                                                      |
| `hub_setting_changes` | Append-only: `id`, `recorded_at`, `revision`, `op` (`seed`, `edit`), `door`, `actor`, `apply_id`, `diff` (JSON `[{key, old, new}]`) |

All three are added to `GLOBAL_TABLES` (`hub/store/schema.py:41`) in one additive revision, applied by
`blizzard hub migrate` (`bzh:manual-migrations`), and `contracts/store/hub.json` is regenerated. A write is the runner's
write ([runner-store.md](./runner-store.md) §A write) over these tables: `If-Match` against `revision`, the whole
resolved configuration validated through the hub's own `parse()` (`hub/config.py`), one revision and one change row per
committed write, `at_default` for an unset. The seed is [settings.md](./settings.md) §Seeding from the file, run by
`blizzard hub serve` after its store check and before `build_services`.

`bzh:configured-record` gains the hub settings record as a kind without retire or enable — there is always exactly one —
keyed by the entry rather than a field, and `bzh:config-read-on-use`'s scope gains the hub-wide record beside the
tenant's.

## Who may change them

A new hub-level `Permission.SETTINGS_ADMIN`, "operate the hub-wide settings every tenant shares", in
`hub/auth/permissions.py` beside `EXPORT_ADMIN`, held only by a hub administrator (`hub_admin_facts`) and, under
`auth.mode = none`, by the implicit operator. No tenant role holds it: a tenant's superuser cannot slow another tenant's
sweeps. Every route below is authorized by `require_hub(Permission.SETTINGS_ADMIN)` (`hub/api/auth_session.py:355`),
reads included, since a tenant's members have no business with another tenant's hub.

## Routes and CLI

| Route                             | CLI                                                | Does                                                                                                                                         |
| --------------------------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /api/admin/settings`         | `blizzard hub settings show [KEY]`                 | The view: as the runner's ([runner-api.md](./runner-api.md) §Views), `in_force` omitted, since every pass and request takes its own snapshot |
| `PATCH /api/admin/settings`       | `blizzard hub settings set KEY VALUE`, `unset KEY` | A write: `set` and `unset` with `If-Match`, refused as the runner's write is                                                                 |
| `GET /api/admin/settings/changes` | `blizzard hub settings changes`                    | The change log, newest first, paginated                                                                                                      |
| `POST /api/admin/settings/apply`  | `blizzard hub settings apply FILE [--dry-run]`     | A declarative apply (§Apply)                                                                                                                 |
| `GET /api/admin/settings/export`  | `blizzard hub settings export`                     | The apply document, as the store stands                                                                                                      |

The routes are described in `openapi/hub.openapi.json` and reach the board through its generated client; the CLI verbs
in `contracts/cli/`. None is on the fleet wire.

## Apply

The hub's settings document is its own, not a section of a tenant's config document, since a tenant's document never
reaches past its tenant (`bzh:config-apply`). It is shaped like `blizzard-hub.toml` — the same tables, the same key
names — and states any of the hub's `pass` keys. An apply sets each key it states and touches nothing else, refuses a
`boot` key with `setting_not_live`, commits in one transaction with door `apply` and one `apply_id`, and answers one
outcome row per key; a dry run is the transaction rolled back. The export writes every entry the store holds, so
applying it back writes nothing.

## Reading per pass

`HubSettingsReader`, in `hub/settings/`, reads `hub_settings.revision` in one indexed read and answers the snapshot,
cached under the revision, so a hub of several processes reads the same settings and a cold one is correct. Each holder
below takes the snapshot at its pass's start, or a request's arrival:

| Held today                                                                                             | Under this epic                                                                                                                                                                                                                                                              |
| ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A sweep's interval, fixed into its `Sweep` at lifespan (`hub/app.py:172`, `_sweep_interval` `:254`)    | `Sweep.run` evaluates the interval before each wait, from the snapshot of the pass before; a write of an interval wakes the waiting sweep, as the runner's driver is woken ([passes.md](./passes.md) §The loop's interval). Whether a sweep is timed or stepped stays `boot` |
| `ActionPassDriver`'s `PassBound` (`hub/composition.py:1514`)                                           | The driver takes `action_pass_seconds` from the snapshot at each pass's start, and the deadline it sets is that pass's                                                                                                                                                       |
| `AnnotationReconciler.full_pass_floor` (`:1708`)                                                       | Computed from `annotation_interval_seconds` at each pass's start                                                                                                                                                                                                             |
| `TraceExportSweep`'s settings, copied at construction (`hub/domain/observability/tracing/sweep.py:60`) | Read from the snapshot at each sweep's start; `TraceReplay` and `TraceStatusReader` read `replay_max_window` per request. Whether the sweep exists at all — an exporter configured — stays `boot`                                                                            |
| `EgressSweep` and `EgressWriterSettings` (`hub/domain/observability/egress/sweep.py:92`, `:1567`)      | The `pass` keys read at each sweep's start; the writer is rebuilt when `max_rows_per_file` or `min_free_bytes` moves. The directory, the format, and whether egress is on stay `boot`                                                                                        |
| `IpThrottle` (`:1686`)                                                                                 | Rebuilt at the first login after its keys move; its buckets start empty, which a throttle loosened or tightened by hand accepts. Turning it off drops it                                                                                                                     |
| `TranscriptCaps`, inside each tenant's cached services (`:1021`, `:1261`)                              | Read from the snapshot per request by the ingest service, so no tenant's cached services hold them                                                                                                                                                                           |
| `close_forge_writes_enabled`, closed over by each tenant's work-source registry (`hub/app.py:618`)     | Read from the snapshot when a closer is handed out                                                                                                                                                                                                                           |
| `produces_mode`, `route_token_mode`, read per request from `app.state.config` (`hub/api/fleet.py:180`) | Read per request from the snapshot                                                                                                                                                                                                                                           |

## The board

The hub's board has no hub-administration page today; its admin shell is a tenant's. It gains one, **Hub settings**,
shown only to a session holding `SETTINGS_ADMIN`: every key of the view with its value, source, and lifetime, an edit
per `pass` key, and the change history, built from the admin shell's record panel, revisions, and field-diff components
(`web/projects/hub/src/app/admin/config-*.ts`).
