# The repository's policy

## The fields

The repository record carries how work lands into it, where `epic:projects` left room for it: as further configured
fields under the record's own verbs and revisions. `repositories` (`hub/store/schema.py`) gains:

```text
repositories
  landing_policy      String   NOT NULL   fast-forward | merge-on-checks | park
  park_timeout_hours  Integer  NULL       how long a parked change may wait; null is the default, 168
```

`RepositoryFields` (`hub/domain/config/repositories.py`) gains `landing_policy: LandingPolicy` and
`park_timeout_hours: int | None`, with `_FIELDS`, `_DIFF_FIELDS`, `RepositoryEdit`, and `merge` following, so a change
to either is a revision of the record with its row in the change log, like any other field. `LandingPolicy` is a closed
`StrEnum` in `hub/domain/config/landing_policies.py`, beside what each policy requires of the forge
([Capabilities](#capabilities)). `validate_fields` refuses an unknown policy, and a `park_timeout_hours` below one or
set on a repository whose policy is not `park`.

A policy belongs to the repository, never to a project or a graph. Every project that links the repository shares it, as
they share its branch. A default on `project_repository_links` waits until one earns its place.

## The order a chunk's repositories land in

A repository may name the repositories it lands after. The order is a relation between records, so it is its own table
rather than a list on the row:

```text
repository_landing_order                 the repositories this one lands after
  tenant_id              String  NOT NULL
  repository_id          String  NOT NULL   FK repositories
  after_repository_id    String  NOT NULL   FK repositories
  PRIMARY KEY (repository_id, after_repository_id)
```

`RepositoryFields` carries the set as `lands_after: frozenset[RepositoryId]`, so the order changes through the same edit
and revision as the policy; `repository_record_store.py` writes the table's rows in the record's update transaction.
Every write resolves each named repository by id or slug through `resolve_repository_reference`, refuses one that does
not exist or is retired (`422 landing_order_unknown`), and refuses an order that would close a cycle anywhere in the
tenant (`422 landing_order_cycle`, naming the cycle by slug). Retiring a repository that another names in its order is
refused with the referrers listed, as retiring a forge in use is.

The order binds only within one chunk: a chunk whose commits reach both a library and its consumer lands the library
first ([landing.md](./landing.md#combining-a-chunks-repositories)). It never holds one chunk behind another, and a
repository it names that a chunk's work does not reach is ignored for that chunk.

## Capabilities

Each policy names the capabilities of `IForgeLanding` it cannot do without, and those it uses when offered:

| Policy            | Requires                                                                                                                 | Uses when offered                |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ | -------------------------------- |
| `fast-forward`    | `refs.read`, `compare`, `refs.advance`, `checks.read`                                                                    | `branches.merge`, `checks.rerun` |
| `merge-on-checks` | `refs.read`, `compare`, `changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, `checks.read` | `checks.rerun`                   |
| `park`            | `refs.read`, `changes.open`, `changes.read`                                                                              | —                                |

`refs.advance` and `branches.merge` are new to the seam ([gitlab.md](./gitlab.md#the-seam-grows)). GitHub offers both;
GitLab offers neither, so a GitLab repository cannot fast-forward, and says so when an operator tries.

The check runs twice. **When a policy is set** — a repository created, its policy edited, or its forge changed —
`ConfigAuthoring` checks the forge's `offered` capabilities against the policy's `requires` and refuses
`422 forge_capability_missing`, naming each missing capability, the policy, and the forge, before anything is written.
The configuration document's apply refuses the entry the same way. **When a visit resolves**, the forges slice's check
before a pass reads the action's requirements, which for `land` are no longer fixed: `IHubAction.requires` becomes
`requires(repository: ResolvedRepository) -> frozenset[Capability]`, and `land` answers its repository's policy's set,
so a policy whose forge stopped offering a capability since it was set is refused naming it, as the forges slice refuses
it. `record-findings` and `garden-deliver` still require nothing.

`ResolvedRepository` carries `landing_policy`, `lands_after`, and `park_timeout_hours`, read with the record at
resolution, so a policy edited mid-delivery takes effect at the visit's next submission and never mid-turn.

## Invariants

`tools/invariants.py` gains `hub:landing-order-acyclic` — no cycle in any tenant's `repository_landing_order`, and no
row naming a retired repository from an enabled one — and widens `hub:repository-references-in-tenant` to the table's
two columns.
