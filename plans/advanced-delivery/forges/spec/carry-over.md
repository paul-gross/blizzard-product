# Carry-over

An installation that predates this slice keeps landing exactly as it did, with every forge it already talks to declared
for it, in one migration and with nothing reconfigured by hand.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, revised after
[blizzard#1106](https://github.com/paul-gross/blizzard/issues/1106)'s, frozen once released (`bzh:frozen-revisions`) and
inside the portable surface (`bzh:sql-portable`). Within every tenant it:

1. **Creates the forges.** It gathers every API origin the tenant's records name — each repository's `forge_api_url`,
   and each GitHub work source's `api_base`, or `https://api.github.com` where that is null — normalized to scheme and
   host with any path kept, and creates one forge per distinct origin: a fresh `frg_<ulid>` id, kind `github`, that
   `api_base`, and the `web_base` of a work source that declared one for the same origin, else null. A forge on
   `api.github.com` is slugged `github` and named `GitHub`; any other takes its web host with dots turned to hyphens as
   its slug (`github-acme-internal`), suffixed `-2` and onwards on collision, and the web host itself as its name. The
   operator changes either afterwards like any other forge's. Each forge gets revision 1, a `forge_lifecycle_facts` row,
   and a `config_changes` row attributed to `migration`.
2. **Points every record at its forge.** Each repository's `forge_id` and each GitHub work source's `forge_id` is set to
   the forge created for its origin, before `repositories.forge_id` is tightened to `NOT NULL` and the old columns
   dropped. A retired record is pointed at its forge like any other, and a forge named only by retired records is
   created retired.
3. **Rebuilds the coordinate constraint** as `(tenant_id, forge_id, owner, repo)`.

No repository or work source changes revision, and no `config_changes` row is written for them: the forge they name is
the address they already held.

## The transition, then retirement

The slice lands in two releases, and it is delivered when the second has.

**The transition release** carries the migration, the forge record and its surfaces, the seam, the callback, and the
ported scripts. The old fields keep working at every door, resolved to forges ([surfaces.md](./surfaces.md#old-fields)),
and a node that declares no `requires_forge` keeps receiving `BZ_FORGE_*` ([landing.md](./landing.md)). Nothing an
operator wrote before the slice stops working: a configuration document kept in their repository applies unchanged, a
CLI invocation in a runbook still runs, and a delivery script of their own still lands. The hub and its packaged scripts
ship together, so a chunk mid-delivery when the hub restarts resumes on the ported script.

**The retirement release** removes the old fields from every door, refusing them `422 field_moved_to_forge`, removes the
old CLI flags, and stops handing `BZ_FORGE_*` to any node. It ships once the `config-old-field-used` and
`forge-token-handed` events have gone quiet on the instances blizzard runs, and its release notes name each removed
field and variable with what replaces it. Both events join the closed `EventLogKind` and its severity map
(`foundation/event_log.py`) as warnings, and both kinds are removed with the fields they report.

Runners are untouched by both: nothing in this slice crosses the runner's wire.

## Proof

The invariant checker's `hub:forge-reference-standing` passes on a carried-over store. A fixture store captured before
the migration, carried over, serves the same repository and work-source documents with `forge` beside the addresses they
held, resolves every chunk's commits to the same repositories, and lands a chunk against the mock forge through the
callback with the same outcomes and markers the token-holding script produced. A configuration document written before
the slice applies to it unchanged, with a warning per old field, and its export applies cleanly after retirement.
