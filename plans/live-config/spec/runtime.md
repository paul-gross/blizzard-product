# How a change takes effect

The store is the only answer to what is configured. No process holds configuration in memory as truth, so a change takes
effect on the next use, with no restart, in one hub process or several.

## Readers

| Reader                                | Today                                                                                                               | After                                                                                                                                                    |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Work-source lookup                    | `WorkSourceEntry.registry(config.work_sources, …)` in `hub/app.py` builds a `WorkSourceRegistry` dict once at start | A store-backed `IWorkSourceRegistry`: `get`, `names`, `annotator`, `closer`, `editor`, and `resolve` read the records on each call                       |
| Delivery's forge settings             | `forge_url`, `forge_token`, `forge_owner`, `base_branch` passed to `build_services` from the environment at start   | Resolved per hub step from the chunk's repositories ([records.md](./records.md) §Resolving a chunk's commits) into `HubEnv` (`hub/delivery/hub_node.py`) |
| Commit resolution for garden delivery | `GitHubCommitResolver` constructed with one forge URL, token, and owner                                             | Resolves the repository row per call                                                                                                                     |
| Forge-status annotation sweep         | Iterates `annotating_names()` of the boot-built registry                                                            | Reads the annotating active sources at the start of each pass                                                                                            |

`IWorkSourceRegistry` keeps its Protocol, so ingest, the close-intent drain, the annotation sweep, and `chunk_views`
change nothing. `close_forge_writes_enabled` stays in the hub's file and is consulted wherever a closer is handed out.

## Read on use

A request reads what it needs when it runs; a sweep reads its records at the start of each pass; a hub step reads its
repositories when it starts. Each read is an indexed single-row read or a read of the active set, the shape
`bzh:live-set-read` sets for a path that repeats.

## Built objects

What a record is turned into is worth keeping: an authenticated `httpx.Client`, a work-source adapter, a resolver. A
process-local `ConfigObjectCache` keeps each built object under `(kind, key, revision, secret_name, secret_revision)`.
Before handing one out it reads the record's and its secret's current revisions in one indexed read, reuses the entry on
a match, and otherwise builds a new object and closes the one it replaces. A decrypted secret value lives only inside a
built object, never in a cache of its own.

The cache holds nothing that is true only in memory. A cold cache is correct, a second hub process reading the same
revisions builds equivalent objects, and nothing announces a change: the first use after it sees the new revision.

## Work in flight

Nothing is pinned to a chunk. A step reads a record when it runs, so an edit made while a chunk is in flight applies to
that chunk's next use of the record. A replaced secret applies on the next forge call, mid-chunk included.

Retirement stops new work only:

- **A retired work source** is excluded from `resolve` and refuses ingest naming it. Items already ingested from it keep
  closing and annotating through it, so `closer` and `annotator` still resolve retired records.
- **A retired repository** refuses resolution for a chunk minted after its retirement fact, and still resolves for a
  chunk minted before it, so work already under way lands.
- **A retired secret** cannot be retired while referenced ([secrets.md](./secrets.md)), so no in-flight use loses its
  credential.

## What stays in the file

These remain `HubConfig` fields or process settings, read at start and changed by a restart: `root`, `db_url`, `host`,
`port`, `trusted_proxies`, `auth`, `runner_auth_mode`, `route_token_mode`, `produces_mode`, `follow_latest`,
`annotation_interval_seconds`, `close_forge_writes_enabled`, `transcripts`, `tracing`, and the hub secret key
([secrets.md](./secrets.md) §The hub key).

## The rule

**`bzh:config-read-on-use`**, in the blizzard-context spoke [api.md](./api.md) §The rule names: a configured record is
read from the store at the use that needs it, and anything built from it is cached under the record's revision. Detect:
a configured record captured at composition time into a long-lived object; a cache of a configured record keyed without
its revision; a configured value read from `os.environ` outside the hub's file and key bootstrap.
