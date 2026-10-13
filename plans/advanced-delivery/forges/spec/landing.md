# Landing on the seam

## The `land` action

The actions slice moved `land_pr_ci` into the hub as the `land` action, speaking GitHub through its own client
(`hub/delivery/actions/internal/github_client.py`). This slice deletes that client. `land` asks `ForgeRegistry` for each
resolved repository's adapter ([seam.md](./seam.md)) and reads and acts through `IForgeLanding` alone: `PullRequest`
becomes `Change`, `Verdict.of` reads a check's `outcome` and a change's `mergeability`, and nothing in the action names
a GitHub route, field, or credential. Its stages, outcomes, and markers are unchanged.

The registry reveals the repository's credential inside the adapter it builds, so the action never handles one. A
`ForgeUnavailable` answers the pass `pending`, as a transient forge state does today; a `ForgeRejected` or
`ForgeNotFound` the action cannot work around answers `failure` with the forge's reason in the pass's log.

`BZ_HUB_GIT_COMMITS` is gone with the actions slice; the action reads the visit's commits, each already resolved to its
repository id.

## What an action requires

`IHubAction` gains `requires: frozenset[Capability]`, the capabilities it cannot do without. `land` requires
`refs.read`, `compare`, `changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, and
`checks.read`; `record-findings` and `garden-deliver` require nothing.

Before a pass, after repository resolution, `HubNodeExecutor` checks every resolved repository's `offered` capabilities
against the action's `requires`. A repository lacking one refuses the visit: it records a `forge-capability-missing`
event naming the capability, the repository, and its forge, writes a `hub-log.forge-capability-missing` artifact, and
routes `failure` before the action runs. The refusal is its own path beside `REFUSAL_EVENT`, which stays keyed by
`RepositoryOutcome`; `forge-capability-missing` joins the closed `EventLogKind`, its severity map (warning), the census,
and `blizzard-context:/domain/operations.md`.

A capability the action can live without is not required. `land` asks `offered(repository)` for `checks.rerun` and,
without it, skips its one re-run and routes the failure as it would after one.

Because the requirement belongs to the action rather than the graph, a graph names no capability, and every graph
version — old or new — is checked the same way.
