# Landing on the seam

## The `land` action

The actions epic moved `land_pr_ci` into the hub as the `land` action, speaking GitHub through `GitHubHandle`
(`hub/delivery/internal/github_handle.py`), one per repository in its `VisitForges`. This epic deletes the handle.
`land` reads and acts through the `RepositoryForge` handed to it for each resolved repository
([seam.md](./seam.md#the-registry)) — its `IForgeRefs`, `IForgeChanges`, and `IForgeChecks` — and nothing else: the
action's own pull-request and verdict types read a `Change` and a `Check`, its routing reads a change's `mergeability`
and a check's `outcome`, and nothing in the action names a GitHub route, field, or credential. Its stages, outcomes, and
the markers it yields are unchanged, and the findings it writes for a failed check still carry the check's link, now its
`details_url`.

The credential is bound inside the adapter the registry built, so the action never handles one. `land` already awaits
its forge on the action thread's loop, so porting it changes how it calls the forge, not where it runs.

## What the forge answers

Today's script answers nearly every forge refusal by waiting for its next poll, and lets the node's poll timeout decide
when waiting has gone on too long; the one refusal it routes is a head it cannot compare, which its head gate counts as
foreign. `land` keeps every one of those answers and changes only what no worker can repair: a credential the forge
rejects, and a repository the forge does not know, now go to a person.

Two rules hold for every operation, before any row below. A `ForgeRejected("auth")` is an escalation to a person with
cause `forge-auth`, and a `ForgeNotFound("repository")` is an escalation with cause `forge-config`, whatever the call
that met it. A `ForgeUnavailable`, after the adapter's own retries, is `pending`, and a `ForgeRateLimited(resume_at)` is
`pending` with its next poll no earlier than `resume_at`, through the pass's next-poll hint. What remains is decided by
the operation:

| Operation                        | Answer, or error                                          | `land` answers                                                                                          |
| -------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `branch_tip` of the base         | `None`, or any other `ForgeRejected`                      | `pending`, as an unread base tip waits today                                                            |
| `branch_tip` of the chunk branch | `None`                                                    | The branch is gone; the closed changes decide, as today                                                 |
| `compare`, in the head gate      | `ForgeNotFound("revision")`, or any `ForgeRejected`       | The head is not admitted, and the chunk routes `failure` with the foreign-head findings                 |
| `compare`, asking if base holds  | `ForgeNotFound("revision")`, or any `ForgeRejected`       | Not held: the landing goes on, as an unreadable comparison is never "nothing to land" today             |
| `changes`, open or closed        | Any `ForgeRejected`                                       | `pending`, as an unreadable list waits today                                                            |
| `open_change`                    | Any `ForgeRejected`                                       | Re-reads the open changes and the comparison: nothing to land, an open change it adopts, else `pending` |
| `change`                         | `ForgeNotFound("change")`, or any `ForgeRejected`         | `pending`; the next pass finds or opens the change afresh                                               |
| `update_from_base`               | `Update(conflict, …)`                                     | `conflict`                                                                                              |
| `update_from_base`               | `Update(refused, …)`, or `ForgeNotFound("change")`        | `pending`, as any answer but a named conflict waits today                                               |
| `merge`                          | Any `ForgeRejected`, or `ForgeNotFound("change")`         | Re-reads the change: merged is landed at its `merged_revision`; still open is `pending`                 |
| `checks`                         | `ForgeNotFound("revision")`, or any `ForgeRejected`       | `pending`, as a degraded check read waits today                                                         |
| `rerun_check`                    | Refused, `ForgeNotFound("check")`, or any `ForgeRejected` | No re-run marker; the failure is re-read and the re-run tried again on the next pass, as today          |

A wrong credential or a repository the forge does not know is a fault in the tenant's configuration, which a worker sent
back down `failure` could never repair, and a pass that only waited would wait out the whole poll timeout before anyone
looked. The escalation names the repository, its forge, and what the forge said, and for `forge-config` it says that the
repository either does not exist or cannot be seen with this credential ([seam.md](./seam.md#failures-and-retries)).
`forge-auth` and `forge-config` join `EscalationCause` (`foundation/escalation_causes.py`), the OpenAPI enum, and the
board's cause sentences.

## What an action requires

The actions epic's `ActionSpec` (`hub/domain/actions/catalogue.py`) gains
`requires: Callable[[ResolvedRepository], frozenset[Capability]]`, the capabilities the action cannot do without for
that repository, declared in the catalogue beside its answers and read by whoever builds a visit, never by the action.
It is a function of the repository from the start, so the `policies` epic changes only what `land`'s answers rather than
where requirements live. In this epic `land`'s answers the same set for every repository: `refs.read`, `compare`,
`changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, and `checks.read`. `record-findings`,
`garden-deliver`, and `pass` answer the empty set. `Capability` is the domain's (`hub/domain/config/capabilities.py`),
so the catalogue names it without importing the seam.

Before a pass, after repository resolution, `HubNodeExecutor` checks every resolved repository's `offered` capabilities
against the action's `requires` for that repository. A repository lacking one refuses the visit: it records a
`forge-capability-missing` event naming the capability, the repository, and its forge, writes a
`hub-log.forge-capability-missing` artifact, and escalates the chunk to a person with cause `forge-capability-missing`
before the action runs, since only an operator can change what a forge offers or what a repository asks of it. The
refusal is its own path beside `REFUSAL_EVENT`, which stays keyed by `RepositoryOutcome`; `forge-capability-missing`
joins the closed `EventLogKind`, its severity map (warning), the census, `EscalationCause`, and
`blizzard-context:/domain/operations.md`. The `flows` epic moves the check, with the building of each `RepositoryForge`,
into its driver's visit build, unchanged.

A capability the action can live without is not required. `land` asks `offered` for `checks.rerun` and, without it,
skips its one re-run and routes the failure as it would after one. The garden's commit check asks `offered` for
`commits.read` and, without it, answers that commit `None`, as a repository it cannot reach answers today.

Because the requirement belongs to the action rather than the graph, a graph names no capability, and every graph
version — old or new — is checked the same way.
