# Landing through the hub

## The forge callback

A `run:` step reaches the forge through the hub, over the same channel its marker writes take. The routes live in
`hub/api/forge_callback.py`, under the chunk, carrying `node_id` and `epoch` in the query string as the marker route
does:

```text
GET  /api/chunks/{chunk_id}/forge/{repository}/branches/{branch}
GET  /api/chunks/{chunk_id}/forge/{repository}/compare?base=&head=
GET  /api/chunks/{chunk_id}/forge/{repository}/commits/{revision}
GET  /api/chunks/{chunk_id}/forge/{repository}/changes?state=&base=&head=
POST /api/chunks/{chunk_id}/forge/{repository}/changes                       {head, base, title, body}
GET  /api/chunks/{chunk_id}/forge/{repository}/changes/{number}
POST /api/chunks/{chunk_id}/forge/{repository}/changes/{number}/update-from-base   {expected_head}
POST /api/chunks/{chunk_id}/forge/{repository}/changes/{number}/merge        {expected_head, method, message}
GET  /api/chunks/{chunk_id}/forge/{repository}/revisions/{revision}/checks
POST /api/chunks/{chunk_id}/forge/{repository}/checks/{check_id}/rerun
GET  /api/chunks/{chunk_id}/forge/{repository}/capabilities
```

`{repository}` is a repository's id or its slug, resolved to the id at the boundary as
[blizzard#1105](https://github.com/paul-gross/blizzard/issues/1105) has every boundary resolve a repository, and never
an `owner/repo`. Each route calls the matching `IForgeLanding` operation ([seam.md](./seam.md)) and answers in the
seam's vocabulary, with wire models in `wire/forge.py`. The seam's errors map to `404 forge_not_found`,
`422 forge_rejected` carrying the reason, `502 forge_unavailable`, and `409 forge_capability_missing` naming the
capability.

## Its authority

Every route depends on `require_marker_authority`: the request carries `X-Blizzard-Marker-Token`, verified against the
`(chunk, node, epoch)` it was minted for. A request without one falls through to `Permission.CHUNK_CONTROL` as the
marker route does, so an operator can drive a landing by hand. A token reaches only the repositories its visit resolved:
`MarkerAuthority.hold_repositories` already pins the visit's resolved repositories, held by id, and a route resolving to
any other repository is refused `403 repository_not_held`. The token is revoked when the visit ends, so the callback
answers nothing after it.

The hub runs the operation with the repository's credential and the script never sees it. The callback is the only door
from a `run:` step to a forge.

## The `run:` environment

`bzh:hub-node-env-contract` (`HubEnv.vars`, `hub/delivery/hub_node.py`) changes:

| Variable                    | Change                                                                                                                                                                                          |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `BZ_FORGE_URL`              | Through the transition, handed only to a node that declares no `requires_forge`; removed when the old fields retire                                                                             |
| `BZ_FORGE_TOKEN`            | As `BZ_FORGE_URL`                                                                                                                                                                               |
| `BZ_FORGE_OWNER`            | As `BZ_FORGE_URL`                                                                                                                                                                               |
| `BZ_HUB_FORGE_CALLBACK_URL` | Added: `{base}/api/chunks/{chunk_id}/forge?node_id=…&epoch=…`, present under the same condition as `BZ_HUB_MARKER_CALLBACK_URL`                                                                 |
| `BZ_HUB_GIT_COMMITS`        | Each entry gains `repository_id`, the id of the record the commit resolved to, and `repository`, its slug as the step starts, beside `repo`; a script addresses the callback by `repository_id` |

A node that declares `requires_forge` has said it speaks the callback, so it is never handed a token, and every shipped
graph's delivery node declares it. An operator's own node that declares nothing keeps receiving the legacy variables
through the transition, so its script keeps landing while it is ported, and each visit that hands them over records a
`forge-token-handed` event in `event_log`. `BZ_HUB_BASE_BRANCH` and every other variable are unchanged.
`SubprocessHubCommandRunner` (`hub/delivery/internal/hub_command_runner.py`) stops passing the daemon's own `BZ_FORGE_*`
through its inherited environment, so a hub still holding a legacy `BZ_FORGE_TOKEN` (`hub/domain/config/legacy_keys.py`)
cannot leak it into a step.

## What a delivery node requires

A hub node may declare `requires_forge`, a list of capabilities its script cannot do without, in the graph's node schema
(`hub/domain/graph/model.py`, validated in `validation.py` against the closed capability set). Before the first step
runs, after repository resolution, the executor checks every resolved repository's `offered` capabilities against it,
and a repository lacking one refuses the visit with the outcome `forge-capability-missing`, naming the capability, the
repository, and its forge. It joins `REFUSAL_EVENT` (`hub_node.py`), the closed `EventLogKind`, and its severity map
beside `repository-not-linked`, and like them routes `failure` before any command runs.

A capability the script can live without is left out of `requires_forge`. The script reads
`GET …/forge/{repository}/capabilities` and takes its own fallback.

## The packaged scripts

`land_common.py` keeps its stdlib-only shape and loses its forge: `forge_request` and `LandRun.api` give way to a
`ForgeCallback` that calls the routes above with the marker token, and `PullRequest` becomes `Change`, reading the
seam's vocabulary. `land_pr_ci.py` and `land_default.py` port onto it with their stages, outcomes, and markers
unchanged. `land_pr_ci.py`'s nodes in the shipped graphs (`basic-development-workflow`, `basic-harness-workflow`,
`advanced-development-workflow`) declare
`requires_forge: [refs.read, compare, changes.open, changes.read,
changes.update-from-base, changes.merge.merge, checks.read]`;
`checks.rerun` stays optional, and without it the script skips its one re-run and routes the failure as it would after
one. `garden_deliver.py` and `review_deliver.py` call no forge and are unchanged.

A graph is immutable once minted, so shipped graphs gain `requires_forge` as new versions; a chunk pinned to an older
version lands through its ported script with no requirement checked.
