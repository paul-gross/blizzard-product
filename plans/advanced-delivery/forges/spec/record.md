# Forge record

## The table

`forges` is a tenant-scoped configured record (`bzh:configured-record`) in `hub/store/schema.py`, shaped as a project
is: an id that every reference holds, a slug people type and may change, and a display name.

```text
forges
  forge_id     String       NOT NULL   frg_<ulid>, minted by the hub, never reused
  tenant_id    String       NOT NULL   the tenant key (multi-tenancy store.md)
  slug         String       NOT NULL   unique within the tenant; may change
  name         String       NOT NULL   the display name; unique nowhere
  kind         String       NOT NULL   one of KNOWN_FORGE_KINDS, checked on write
  api_base     String       NOT NULL   the forge's API origin (https://api.github.com)
  web_base     String       NULL       its web origin; derived from api_base when null
  revision     Integer      NOT NULL
  created_at   UtcDateTime  NOT NULL
  created_by   String       NOT NULL   the acting user id, or `migration`
  PRIMARY KEY (forge_id)
  UNIQUE (tenant_id, slug)
  UNIQUE (tenant_id, api_base)

forge_former_slugs                      mirrors project_former_slugs
  tenant_id, slug, forge_id, left_at
  PRIMARY KEY (tenant_id, slug)

forge_lifecycle_facts                   append-only; mirrors repository_lifecycle_facts
  id, tenant_id, forge_id, retired Boolean, set_at, set_by
```

`IdPrefix` (`foundation/ids.py`) gains `FORGE = "frg"`. `KNOWN_FORGE_KINDS = {"github"}` lives in
`hub/domain/config/forges.py` with `ForgeFields(slug, name, kind, api_base, web_base)`, `ConfiguredForge`, `ForgeEdit`,
and the read and write repository Protocols (`IReadForgeRepository`: `get`, `get_many`, `list_all`;
`IWriteForgeRepository`: `create`, `update`, `record_lifecycle`), patterned on `repositories.py` and implemented in
`hub/store/internal/forge_record_store.py`. Two forges never share an API origin, so one server is one record.
`web_base`, when null, derives through `hub/forge/web.py::web_origin` as the work-source factory derives it today; the
derivation moves off `GithubEntry` and onto the forge.

A forge is resolved at every boundary as a project is (`hub/domain/projects/resolve.py::resolve`): a `frg_<ulid>` value
by id, otherwise a current slug, otherwise a former slug until another forge of the tenant claims it. Nothing past a
boundary handles a slug, so changing a forge's slug touches no record that names it.

A forge holds no credential. Its kind decides which adapter speaks to it and which capabilities it offers
([seam.md](./seam.md)).

## References

`repositories.forge_api_url` is replaced by `repositories.forge_id`, `NOT NULL`, a foreign key onto `forges.forge_id`
within the tenant. The coordinate constraint `uq_repositories_tenant_coordinate` becomes
`(tenant_id, forge_id, owner, repo)`. `RepositoryFields` swaps `forge_api_url` for `forge_id`.

`work_sources.api_base` and `work_sources.web_base` are replaced by `work_sources.forge_id`, nullable, with the same
foreign key. A work source whose `provider` is a forge kind must name a forge of that kind; a provider that is not a
forge must name none. Both are checked in `work_sources.validate_fields`. `uq_work_sources_tenant_provider_locator` is
unchanged: a GitHub locator is `owner/name` and unique per provider, as before, and the forge it lives on does not
change that.

Each referring record keeps its own `secret_name`. The credential belongs to the use, never to the forge.

## Retirement

A forge that an enabled repository or enabled work source names is refused retirement, `409 forge_in_use`, listing the
referrers, the way a secret in use is refused today. The referrer lookup joins `store/internal/secret_referrers.py`'s
`_REFERRING` pattern in a `forge_referrers.py` beside it. A retired forge can still be read, and records retired while
naming it keep naming it; enabling such a record is refused while its forge is retired. A retired forge keeps its slug,
so no other forge can take it while the retired one could be enabled again.

Resolution follows: `resolve_chunk_repositories` (`hub/domain/config/repositories.py`) matches a commit's `forge` origin
host against the named forge's web host and API host (`_forge_host` reads the forge row instead of `forge_api_url`), and
`ResolvedRepository` carries `forge_id` in place of `forge_api_url`. `repositories-disagree` compares `forge_id`,
`owner`, `base_branch`, and `secret_name`; lifting it is the `policies` slice's.

## The change log

`RecordKind` (`hub/domain/config/changes.py`) gains `forge`, with its transitions in `FIELDED_RECORD_TRANSITIONS`, and a
forge's change rows are keyed by its id as a repository's are once it carries one. `config_changes.project_id` stays
null for it, as for every tenant-wide record. Every other enumeration of record kinds gains it too:
`config_revisions_store.py`'s `_REFERRING`, `config_apply_store.py`'s dispatch, `hub/cli/config.py`'s `_KINDS`, and the
generated `RecordKind` in the web client.

## Invariants

`tools/invariants.py` gains `hub:forge-reference-standing`: no enabled repository or work source names a retired forge,
every work source whose provider is a forge kind names a forge of that kind, and no former slug is also a current slug
in its tenant. `hub:landed-repository-linked` is unchanged.
