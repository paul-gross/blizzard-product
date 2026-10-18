# Forge record

## The table

`forges` is a tenant-scoped configured record (`bzh:configured-record`) in `hub/store/schema.py`, shaped as projects,
repositories, and work sources already are: an id that every reference holds, a slug people type and may change, and a
display name. Repositories (`repo_<ulid>`) and work sources (`wsrc_<ulid>`) already have that shape, through a shared
identity kernel, and a forge is born into it rather than migrated onto it.

```text
forges
  forge_id     String       NOT NULL   frg_<ulid>, minted by the hub, never reused
  tenant_id    String       NOT NULL   the tenant key (multi-tenancy store.md)
  slug         String       NOT NULL   unique within the tenant; may change
  name         String       NOT NULL   the display name; unique nowhere
  kind         String       NOT NULL   one of KNOWN_FORGE_KINDS, checked on write
  api_base     String       NOT NULL   the forge's API origin, canonical (https://api.github.com)
  web_base     String       NULL       its web origin; derived from api_base when null
  revision     Integer      NOT NULL
  created_at   UtcDateTime  NOT NULL
  created_by   String       NOT NULL   the acting user id, or `migration`
  PRIMARY KEY (forge_id)
  UNIQUE (tenant_id, slug)
  UNIQUE (tenant_id, api_base)

forge_former_slugs                      mirrors repository_former_slugs
  tenant_id, slug, forge_id, left_at
  PRIMARY KEY (tenant_id, slug)
  FOREIGN KEY (forge_id) → forges.forge_id

forge_lifecycle_facts                   append-only; mirrors repository_lifecycle_facts
  id, tenant_id, forge_id, retired Boolean, set_at, set_by
```

`IdPrefix` (`foundation/ids.py`) gains `FORGE = "frg"`. `hub/domain/config/forges.py` is patterned on `repositories.py`
as it now stands, and holds:

- `KNOWN_FORGE_KINDS = {"github"}`, and beside it `FORGE_KIND_CAPABILITIES`, the `Capability` set
  (`hub/domain/config/capabilities.py`) each kind offers ([seam.md](./seam.md#capabilities)).
- `canonical_origin(url)`, the one form an address is stored and compared in. Today nothing normalizes an address —
  `forge_api_url`, `api_base`, and `web_base` are stored as they were typed; a repository's `validate_fields` checks
  only that its address has an `http` or `https` scheme and a host, and a work source's checks neither of its addresses
  at all, so a stored `api_base` may lack a scheme or be an empty string, which the work-source factory reads as
  github.com — so the rule is stated whole:
  - Surrounding whitespace is trimmed; whitespace inside the address is refused.
  - The scheme is `http` or `https`, lowercased; an address with any other scheme, or none, is refused, as is one with
    no host.
  - The host is lowercased and one trailing dot is dropped. An internationalized host is stored in its ASCII form
    through Python's built-in `idna` codec (IDNA 2003), named so that the hub and the frozen migration encode alike, and
    an IPv6 literal keeps its brackets, lowercased. A host carrying a percent-escape is refused.
  - A scheme's default port, 80 or 443, is dropped, and so is an empty port (`host:`); any other port is kept.
  - The path keeps its case, since a forge's paths are case-sensitive. Its percent-escapes are normalized as RFC 3986
    asks — hexadecimal digits uppercased, and an escape of an unreserved character decoded — and trailing slashes are
    stripped; an empty path stays empty.
  - An address carrying a user or password, a query, or a fragment is refused rather than cleaned. The hub authenticates
    with a header, never with an address, and a credential written into one would otherwise sit in the change log in the
    clear.

  So `http://127.0.0.1:8123/` is stored as `http://127.0.0.1:8123`, `https://API.GitHub.com:443/` and
  `https://api.github.com.` and `https://api.github.com` are one origin, and `https://ghe.acme.internal/api/v3` and
  `https://ghe.acme.internal/API/v3` are two. Where an address is optional, an empty or whitespace-only value is no
  value, as the work-source factory's `or` already reads it. The rule is pinned by one table of cases,
  `tests/fixtures/canonical_origin_cases.json` — every bullet above, an empty port, a trailing dot, an IDNA host, an
  escaped path, and each refusal — read by both `canonical_origin`'s unit test and the migration's test of its own
  frozen copy, so the two can never disagree about an address.
- `derived_web_origin(kind, api_base)`, the web origin a forge of that kind implies from its API origin, which moves
  here from `hub/forge/web.py::web_origin` so the domain can derive a forge's web host without importing the forge edge;
  for `github` it is the derivation the work-source factory makes today, and `hub/forge/web.py` calls it.
- `ForgeSlug(Slug)` and `ForgeName(DisplayName)`, whose grammars come from `hub/domain/kernel/slugs.py`, and
  `FORGE_KIND = SluggedKind(IdPrefix.FORGE, ForgeId, ForgeSlug.of)`.
- `ForgeFields(api_base, web_base)`, the mutable fields only; the slug, name, and kind sit on the record, as
  `RepositoryFields` leaves the slug and name out. A forge's kind never changes once it is created, since every adapter
  built from it and every work source that names it was chosen for that kind.
- `ConfiguredForge(forge_id, slug, name, kind, fields, …)` with `new`, `edit`, `set_retired`, `ref()`, and
  `slug_left_for()`, and `ForgeEdit`, which carries a new slug or name beside the fields.
- `IReadForgeRepository` (`get`, `get_many`, `list_all`, `get_by_slug`, `get_by_former_slug`, `resolve`),
  `resolve_forge_reference`, and `IWriteForgeRepository` (`create`,
  `update(record, *, from_revision, left_slug,
  change)`, `record_lifecycle`).

The store is `hub/store/internal/forge_record_store.py`, which claims and releases slugs through
`SlugTables(forges, forge_former_slugs, "forge_id", ForgeSlugTaken)` (`hub/store/internal/former_slugs.py`). Every write
stores `api_base` and `web_base` through `canonical_origin`, and two forges never share an API origin, so one server is
one record: a create or edit whose origin another forge of the tenant holds is refused `ForgeOriginTaken`, naming that
forge, as `RepositoryCoordinateTaken` refuses a taken coordinate. `web_base`, when null, derives through
`derived_web_origin`; the derivation moves off `GithubEntry` and onto the forge.

A forge's row is locked as a secret's is. `lock_forge_row(conn, forge_id)` in the forge store, patterned on
`lock_secret_row` (`hub/store/internal/secret_store.py`), is the first statement after the scoped write's tenant-row
lock in a forge's retire and enable and in every repository or work-source write that names a forge, so a retire and an
enable racing on one forge wait for each other's commit rather than leaving an enabled record on a retired forge;
`tests/test_forge_row_lock.py` proves it as `tests/test_secret_row_lock.py` proves the secret's. `IWriteForgeRepository`
joins `_CONFIGURED_WRITE_NAMES` and `hub/domain/config/forges.py` joins `_CONFIGURED_WRITE_HOMES`
(`tests/test_layering.py`), so only the authoring service writes a forge, as only it writes the other configured
records. The forge's effective web origin — its own `web_base`, else the derived one — now serves every record that
names it. Today a repository's branch links always derive from its `forge_api_url`, while a work source on the same
server may declare a `web_base` of its own; once both name one forge, a `web_base` set on that forge moves the
repository's branch links with it.

A forge is resolved at every boundary through the shared kernel every slugged kind uses
(`hub/domain/kernel/id_or_slug.py::resolve`): a `frg_<ulid>` value by id, otherwise a current slug, otherwise a former
slug until another forge of the tenant claims it. Nothing past a boundary handles a slug, so changing a forge's slug
touches no record that names it.

A forge holds no credential. Its kind decides which adapter speaks to it and which capabilities it offers
([seam.md](./seam.md)).

## References

`repositories.forge_api_url` is replaced by `repositories.forge_id`, `NOT NULL`, a single-column foreign key onto
`forges.forge_id`, as every id reference is (`fk_project_repository_links_repositories`); that the forge is of the
repository's own tenant is an invariant's to check ([Invariants](#invariants)). The coordinate constraint
`uq_repositories_tenant_coordinate` becomes `(tenant_id, forge_id, owner, repo)`, and so does the store's own check
before it (`_coordinate_holder`, `RepositoryCoordinateTaken`, in `repository_record_store.py`). `RepositoryFields` swaps
`forge_api_url` for `forge_id`.

`work_sources.api_base` and `work_sources.web_base` are replaced by `work_sources.forge_id`, `NOT NULL`, with the same
kind of foreign key. Every provider a work source can name today is a forge — `KNOWN_WORK_SOURCE_PROVIDERS` is
`{"github"}`, and the built-in hub source is no row at all — so every row names one; a provider that is not a forge,
should one arrive, relaxes the column in the change that adds it. A work source must name a forge whose kind is its
`provider`. Whether it does needs the forge, so the work-source store's write checks it, as the repository store checks
that a secret is available (`RepositorySecretUnavailable`).

`uq_work_sources_tenant_provider_locator` stays `(tenant_id, provider, locator)` through the transition release. A
GitHub locator is `owner/name`, and `acme/app` on github.com and `acme/app` on an Enterprise server are in truth two
trackers, but letting an installation configure both side by side would leave a store the transition's downgrade could
not walk back, since the column that told them apart is the one being dropped
([carry-over.md](./carry-over.md#the-migration)). The constraint widens to `(tenant_id, provider, forge_id, locator)` in
the retirement release, which breaks the old fields anyway, and the store's own check before it (`_locator_holder`,
`WorkSourceLocatorTaken`, in `work_source_record_store.py`) and the legacy import's locator set
(`hub/domain/config/carry_over.py`) key on the forge with it.

Each referring record keeps its own `secret_name`. The credential belongs to the use, never to the forge.

## Retirement

A forge that an enabled repository or enabled work source names is refused retirement, `409` with the coded detail
`{"code": "forge_in_use", "message": …}` and the referrers listed as `RecordRef`s, as `SecretView.references` lists a
secret's — a secret in use is refused today with a plain-string detail, which a client cannot tell from any other
conflict. The referrer lookup joins `store/internal/secret_referrers.py`'s `_REFERRING` pattern in a
`forge_referrers.py` beside it. A retired forge can still be read, and records retired while naming it keep naming it.
Enabling such a record is refused while its forge is retired: `ConfigAuthoring`'s `enable_repository` and
`enable_work_source` read the named forge and refuse `422` with the coded detail `forge_retired`, naming it, as enabling
a repository whose secret has since been retired is refused `422` today (`hub/api/repositories.py`), and the
configuration document's apply, which re-enables a retired record it declares, refuses the entry the same way. A retired
forge keeps its slug and its API origin, so no other forge can take either while the retired one could be enabled again.

Resolution follows. `_candidates` (`hub/domain/config/repositories.py`), which `resolve_chunk_repositories`,
`resolve_origin_record`, and `resolve_repository` all match through, compares a commit's `forge` origin host against the
named forge's web host and API host, so each of the three is handed the tenant's forges beside its repositories and
`_forge_host` reads the forge row instead of `forge_api_url`, taking the web host from the forge's own `web_base` or
`derived_web_origin`, both in the domain. Every caller that reads `list_all` on the repositories for one of them reads
`IReadForgeRepository.list_all(include_retired=True)` beside it, retired forges included as retired repositories are,
and passes both. `ResolvedRepository` carries `forge_id` in place of `forge_api_url`. `ResolvedRepository.target` and
`RepositoryResolution.agreed()` lost their one caller, the hub node's step environment, with the actions epic, and are
deleted rather than carried over. Each commit pointer already resolves to its own repository and lands against its own
forge, owner, base branch, and credential — the refusal `repositories-disagree` that once kept one chunk on one forge is
gone — so naming the forge changes what a target carries, not which targets a chunk may have. `hub/api/chunk_views.py`,
which builds branch links through `hub/forge/web.py::branch_url`, reads the forges of the repositories it links in one
read.

## The change log

`RecordKind` (`hub/domain/config/changes.py`) gains `forge`, with its transitions in `FIELDED_RECORD_TRANSITIONS`, and a
forge's change rows are keyed by its id, as projects', repositories', and work sources' are. `config_changes.project_id`
stays null for it, as for every tenant-wide record. Every other enumeration of record kinds gains it too:
`config_apply_store.py`'s dispatch, `_REFUSALS`, and `_taken`; `hub/api/config.py`'s `_SCHEMAS`, `_labels`, and
`_key_of`, with a `_FORGE_KEYED` beside `_REPOSITORY_KEYED`; `hub/cli/config.py`'s `_KINDS`; and, in the web client, the
generated `RecordKind`, `SURFACE_SEGMENT` (`admin/config-links.model.ts`), `AdminRecordKind` and `CLI_CHANGE_PATH`
(`admin/config-cli.model.ts`), `admin.routes.ts`, and the tab counts in `admin-shell.ts`.

A repository's and a work source's diff (`_DIFF_FIELDS`) swap their address fields for `forge`, recorded as the forge's
id, as every reference is held, and shown with the forge's current slug and name when the change log is read. That
showing is new: `FieldChangeView` (`wire/config.py`) is `(field, old, new)` today, and `_labels` (`hub/api/config.py`)
labels only the record a change row is keyed by. `FieldChangeView` gains `old_label` and `new_label`, optional, filled
for a `forge` field from one read of the forges a page of changes names, beside `_labels`' reads, while `old` and `new`
keep the ids. A row written before the epic keeps the field names it was written with — `forge_api_url`, `api_base`,
`web_base` — and is shown as written: the change log is a record of what changed, and it never rewrites its past to the
present shape.

A forge holds no secret, so it takes no entry in `config_revisions_store.py`'s `_REFERRING`, which joins each kind's
`secret_name`. What it changes instead is the cache key of every record that names it: `ConfigRevisions`
(`hub/domain/config/revisions.py`) gains `forge_revision`, read by joining `forges` for the repository and work-source
kinds, and `CacheKey` (`hub/live_config/__init__.py`) carries it, so an edit to a forge rebuilds every adapter built
from it — the landing seam's ([seam.md](./seam.md#the-registry)) and `StoreWorkSourceRegistry`'s alike.

## Invariants

`tools/invariants.py` gains two checks and widens one:

- `hub:forge-reference-standing` — no enabled repository or work source names a retired forge, and every work source
  names a forge whose kind is its provider.
- `hub:forge-references-in-tenant` — `repositories.forge_id`, `work_sources.forge_id`, `forge_former_slugs`,
  `forge_lifecycle_facts`, and the forge's change rows each name a forge of their own tenant, as
  `hub:repository-references-in-tenant` and `hub:work-source-references-in-tenant` check their kinds.
- `hub:slug-not-current-and-former` gains forges in `SlugNotCurrentAndFormer.KINDS`.

`hub:landed-repository-linked` is unchanged.
