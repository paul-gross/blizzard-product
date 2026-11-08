# The urgency record and the tenant's settings

## What an urgency holds

An urgency is a tenant-wide configured record (`bzh:configured-record`), held as a routine is: a surrogate id every
reference holds, a slug fixed at create, a display name free to change, a `revision`, and a retired brake kept as
append-only lifecycle facts (`bzh:facts-not-status`). Unlike the other tenant-wide records, a forge or a repository, its
slug never changes, so it has no former slugs to resolve; and it departs from the shared verb set where the
configuration spoke's entry for it says — its set has an order, and its retire takes a body and can be refused while
open work carries it.

```yaml
slug: urgent
name: Urgent # the slug when absent
color: red # a palette name or #rrggbb; picked when absent
emoji: "🔥" # or icon, or neither
```

| Field   | Type   | Default  | Meaning                                                                                                                                                                 |
| ------- | ------ | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `slug`  | string | required | The handle typed in the CLI, the config document, and URLs, in the slug grammar every configured record shares; unique in its tenant, retired urgencies included; fixed |
| `name`  | string | the slug | What a person reads; free text, never blank, need not be unique, never resolved                                                                                         |
| `color` | string | picked   | The pill's fill, in the grammar a project's color takes: a name from the shared palette or a `#rrggbb`, stored lowercase                                                |
| `emoji` | string | absent   | The pill's mark, when it carries one ([§The mark](#the-mark))                                                                                                           |
| `icon`  | table  | absent   | The pill's mark as an image the operator supplies: `media_type` and base64 `data` ([§The mark](#the-mark))                                                              |

An urgency created without a color is given the palette color the tenant's live urgencies use least, picked as a new
project's is, so a set built from the CLI without a thought for color still reads as distinct pills.

## Validation

A create, an edit, and an apply refuse, naming the field:

- a slug outside the grammar, or one an urgency of the tenant already holds, live or retired — `422 urgency_slug_taken`,
  whose detail says when the holder is retired, so an operator who meant to bring it back is pointed at enable;
- a blank `name`, or an explicit `null` on `name` or `color`;
- a `color` that is neither a palette name nor a `#rrggbb`;
- an `emoji` that is blank, longer than 16 code points, or carries whitespace, a control character, or an ASCII letter —
  so a word cannot pass for a mark. An ASCII digit, `#`, or `*` is admitted only as the base of a keycap sequence, so
  `1️⃣` is a mark and `12` is not;
- an `icon` whose bytes are neither PNG nor WebP by their signature, or exceed 32 KiB once decoded. SVG is refused
  outright, since an image the board draws must not be able to carry script;
- a write that would leave both `emoji` and `icon` set — `422 urgency_mark_conflict`. Switching from one to the other
  clears the old one with an explicit `null` in the same edit, which the board's form does on the operator's behalf.

## The mark

The pill shows one mark: the emoji when there is one, the icon when there is one, and the name otherwise
([surfaces.md](./surfaces.md) §The pill). An icon is stored on the record, with its media type and a digest of its
bytes, and served by `GET /api/urgencies/{key}/icon` with that media type, the digest as its `ETag`, and
`Content-Security-Policy: default-src 'none'`, so a browser caches it across every card and never runs it. An urgency's
views carry the icon's media type and digest, never its bytes; only the config document's export carries the bytes.

## The set's order

The urgencies of a tenant form one order, most urgent first. It is kept as append-only position facts, as a project's
landing order is: a position fact is not a revision of the urgency it places, and only a change of place appends one.
The newest fact per urgency decides; ties break on the slug. Because the order decides claim order for the whole tenant,
a reorder that moves anything also appends one change row, kind `urgency_order`, so the history shows who reordered it.

- **Create** lands a new urgency at the least urgent end. A create stating `after` lands it just after the urgency named
  there, and `after: null` lands it first.
- **`PUT /api/urgencies/order`** sets the whole live order from a list naming every live urgency once, and is refused
  `422 urgency_order_incomplete` when it leaves one out, names a retired one, or names one twice.
- **`POST /api/urgencies/order/position`** moves one urgency to just after another, or first under `after: null`.

Each reorder takes the set's keyed lock, re-reads the order under it (`bzh:write-reestablishes-read`), and publishes the
queue's change after its commit (`bzh:publish-after-commit`), since claim order moves with it. A retired urgency keeps
its standing position fact, so an urgency revived returns to the place it left.

## The tenant's settings

The tenant's settings are a singleton configured record, one per tenant, keyed by the tenant's id, with a `revision` and
a change row on every write under the new record kind `tenant_settings`. It takes show and edit and nothing else: a
tenant always has its settings, so they are neither created nor retired, and its routes carry no key, since the tenant
the request resolved is the key. Both are departures the configuration spoke's entry for it records. This epic gives it
one field.

| Field             | Type                   | Meaning                                                                                                      |
| ----------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------ |
| `default_urgency` | an urgency, id or slug | The urgency every new chunk takes when nothing names another ([chunk.md](./chunk.md) §The chain); never null |

`GET /api/settings` answers the record; `PATCH /api/settings` edits it as every configured record is edited, a sparse
merge with `If-Match` on its `revision`. An edit naming an urgency the tenant does not hold is refused
`422 urgency_unknown`, one naming a retired urgency `422 urgency_retired`, and an explicit `null` is refused. The edit
takes the named urgency's lock and confirms it live, so the default can never land on an urgency being retired. Changing
the default moves no chunk: it decides only what is minted from then on, and which urgency the board treats as ordinary.

## Retiring, with the work moved

Retire takes a body: `{ "move_to": <urgency> }`, by id or slug. It is refused, in this order:

1. `409 urgency_is_default` when the urgency is the tenant's default. The detail says to point the default elsewhere
   first.
2. `409 urgency_in_use` when open chunks carry it and the body names no `move_to`. The detail counts them, so a client
   can ask the operator where they go.
3. `422 urgency_unknown` or `422 urgency_retired` when `move_to` names no urgency of the tenant, or a retired one, and
   `422 urgency_move_to_self` when it names the urgency being retired.

A retire that clears all three is one unit of work (`bzh:unit-of-work`) declaring, as every hub unit does, its locks
once: the settings record's and those of the retiring urgency and of `move_to`. Under them it re-reads the default
(`bzh:write-reestablishes-read`), then writes together the retired fact with the urgency's `revision` and change row,
whose diff names `move_to` and how many chunks moved; every open chunk still carrying the retiring urgency moved to
`move_to`, its rank untouched; and every routine default and chore override naming it cleared, each with its record's
own `revision` and change row — the cadence's, for a chore. A retire with no open chunk behind it moves nothing.

No chunk needs locking: every write that sets a chunk, a routine, a chore, or the default to an urgency declares that
urgency's lock and confirms it live ([chunk.md](./chunk.md) §Setting an urgency), so nothing can take up the retiring
urgency while the retire holds it. `blizzard:postgres-uow-concurrency` proves a retire racing a mint, a chunk edit, and
a settings edit. After the commit, each moved chunk publishes its change with cause `edited`, and the queue its change
once.

Finished chunks are not open, so they keep the urgency they ran under, retired or not, and every read that shows them
resolves it as it resolves any other.

Retiring needs `config:edit`, as every configured write does; a retire that moves open work needs `chunk:control` as
well, the permission editing a chunk already asks for.

## Enabling

Enable is the revival the plan names. It is never refused for want of anything: the urgency's slug can belong to no
other urgency, so it comes back meaning what it meant, at its standing place in the order, choosable again. It moves no
chunk back: the work it once carried went where the operator sent it.

## The tables

Four tables, each tenant-keyed with a tenant-led index (`bzh:tenant-keyed-table`):

- **`urgencies`** — `urgency_id` (`urg_<ulid>`, hub-unique), `slug`, `name`, `color`, `emoji` (nullable),
  `icon_media_type`, `icon_data`, and `icon_digest` (each nullable), `created_at`, `created_by`, `revision`,
  `tenant_id`; unique on `(tenant_id, slug)`. Because retirement is a fact beside the row, the uniqueness covers retired
  urgencies with no further rule.
- **`urgency_lifecycle_facts`** — `id`, `urgency_id`, `retired`, `set_at`, `set_by`, `tenant_id`; newest wins.
- **`urgency_positions`** — `id`, `urgency_id`, `position` (a float, lower more urgent), `set_at`, `set_by`,
  `tenant_id`; newest per urgency wins.
- **`tenant_settings`** — `tenant_id` (the key), `default_urgency_id`, `revision`, `updated_at`, `updated_by`.

The id prefix `urg` joins the hub's id prefixes, and `urgency`, `urgency_order`, and `tenant_settings` join the record
kinds a change row names. A change row of any of them names no project, as a forge's does. The store's contract document
is regenerated, and tenant teardown and the store's read census learn the four tables.

## The domain package

The urgency is a domain package of its own, declared in the domain's layering below the chunk, the work items, the
garden, and the cadence, which read it, and below the config apply, which composes it (`bzh:domain-package-layers`). Its
model holds the record, the mark's validation, the set's order, and the retire's refusals as pure values; its services
hold the verbs, the reorders, the settings record, and the retire's unit of work (`bzh:domain-orchestration-split`). It
answers one question for everything above it: the tenant's live urgencies in order, with the default marked, read once
per request or pass as a read of the active set (`bzh:live-set-read`, `bzh:config-read-on-use`).

## The verbs

| Route                                      | Verb     | Notes                                                           |
| ------------------------------------------ | -------- | --------------------------------------------------------------- |
| `POST /api/urgencies`                      | create   | `201` with the urgency's view; `after` optional                 |
| `GET /api/urgencies`                       | list     | the live set in order; retired ones under `?retired=true`       |
| `GET /api/urgencies/{key}`                 | show     | `{key}` is the id or the slug                                   |
| `PATCH /api/urgencies/{key}`               | edit     | `If-Match` on `revision`, `409` when stale; `slug` refused      |
| `POST /api/urgencies/{key}/retire`         | retire   | [§Retiring, with the work moved](#retiring-with-the-work-moved) |
| `POST /api/urgencies/{key}/enable`         | enable   |                                                                 |
| `GET /api/urgencies/{key}/icon`            | icon     | `404` when the urgency has none                                 |
| `PUT /api/urgencies/order`                 | order    | the whole live order                                            |
| `POST /api/urgencies/order/position`       | position | one urgency moved                                               |
| `GET /api/settings`, `PATCH /api/settings` | settings | [§The tenant's settings](#the-tenants-settings)                 |

An urgency's view carries its id, slug, name, color, emoji, the icon's media type and digest, whether it is retired, its
place in the order, whether it is the default, its revision, and how many open chunks carry it, counted in one grouped
read beside the list. Every write needs `config:edit`; every read needs `fleet:view`.

## In the config document

The tenant's config document gains three sections, in its current version, each an optional addition a document written
today leaves out:

```yaml
urgencies:
  - slug: urgent
    color: red
    emoji: "🔥"
  - slug: standard
    color: slate
  - slug: background
    name: Whenever there is room
    color: "#5b6b7a"
urgency_order: [urgent, standard, background]
settings:
  default_urgency: standard
```

An `urgencies` entry is keyed by its slug and reconciled as every entry is — created when absent, a sparse edit of the
fields it states when present — with one departure the configuration spoke records: an entry naming a retired urgency is
refused `urgency_retired` rather than enabled, since reviving an urgency is an act an operator takes on purpose. The
same refusal meets any reference to a retired urgency anywhere in the document, and `urgency_unknown` meets one to an
urgency neither stored nor declared there. A new urgency the document creates lands at the least urgent end, in document
order. `urgency_order`, when stated, sets the whole live order and must name every urgency live once the apply has run,
exactly once, else the apply is refused `urgency_order_incomplete`; omitted, the order stands.
`settings.default_urgency`, when stated, sets the default.

The apply reconciles `urgencies`, then `urgency_order`, then `settings`, all before the routines and cadences that may
name them, so one document can declare a `background` urgency and the gardening routine that enters there. Export writes
every live urgency, an icon as its media type and base64 bytes, the order, and the settings, so applying an export back
writes nothing. The `urgencies` and `settings` sections' schemas are served at `GET /api/config/schema/urgencies` and
`GET /api/config/schema/settings`.
