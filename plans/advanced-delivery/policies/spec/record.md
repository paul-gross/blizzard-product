# The repository's policy, and the project's order

## The fields

The repository record carries how work lands into it, where `initiative:projects` left room for it: as further
configured fields under the record's own verbs and revisions. `repositories` (`hub/store/schema.py`) gains:

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

A chunk belongs to exactly one project — `chunks.project_id` is set once, at mint — and lands only in the repositories
that project links, so the order its repositories land in is the project's to say. Two projects that link the same
library and consumer may order them differently, and each chunk lands by its own project's order.

The order is a position on each of the project's repository links, set by dragging the project's repositories into the
order they should land in. It is kept the way the ready queue keeps a chunk's position, as an append-only fact whose
newest row wins, rather than as a field of the link:

```text
project_repository_positions             a repository's place in its project's landing order
  id              Integer   PK, autoincrement
  project_id      String    NOT NULL
  repository_id   String    NOT NULL
  position        Float     NULL         lower lands first; null is unranked
  set_at          UtcDateTime NOT NULL
  set_by          String    NOT NULL
  tenant_id       String    NOT NULL     FK tenants
  FOREIGN KEY (project_id, repository_id) → project_repository_links
  INDEX (tenant_id, project_id)
```

The order is strict. A visit's repositories sort by position, lower first; a repository with no position, or whose
newest fact is null, sorts after every positioned one; and any two that would tie — unranked ones among themselves, or
two that share a number — sort by their slug, so a project that never orders anything lands `blizzard`, then
`blizzard-context`, then `blizzard-mock`. A position is a float so that a drag writes one fact, the midpoint between the
two repositories it now sits between, as `queue_positions` does.

The fact names a link, not a repository: a link's own facts keep it in place, so unlinking a repository leaves its
position row in place and ignored, and relinking it brings that position back. Writing a position for a repository the
project does not link is refused `422 repository_not_linked`. A position write is not a revision of the link and writes
no `config_changes` row; it carries who set it, as a link's facts do. `hub/store/internal/project_position_store.py`
writes and reads the facts, and the project's repository-link reads carry each link's current position.

The order binds only within one chunk: a chunk whose commits reach both a library and its consumer lands the one ranked
first before the other ([landing.md](./landing.md#combining-a-chunks-repositories)). It never holds one chunk behind
another, and a repository it ranks that a chunk's work does not reach is ignored for that chunk.

`LandingProject` (`hub/delivery/hub_node.py`), read as a visit starts, gains `positions`, each linked repository's
current position or `None`, and resolution copies each resolved repository's onto `ResolvedRepository.landing_position`,
so the order a visit lands by is the project's order as its links stand when the visit began.

## Capabilities

Each policy names the capabilities it cannot do without, and those it uses when offered:

| Policy            | Requires                                                                                                                 | Uses when offered                              |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------- |
| `fast-forward`    | `refs.read`, `compare`, `refs.advance`, `checks.read`                                                                    | `branches.merge`, `refs.create`, `refs.delete` |
| `merge-on-checks` | `refs.read`, `compare`, `changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, `checks.read` | `checks.rerun`                                 |
| `park`            | `refs.read`, `compare`, `changes.open`, `changes.read`                                                                   | —                                              |

`refs.advance`, `branches.merge`, `refs.create`, and `refs.delete` are new to the seam
([seam.md](./seam.md#the-seam-grows)). GitHub offers all four; a forge kind that offers none cannot fast-forward, and
says so when an operator tries. `fast-forward` without the three it uses still lands a revision that is already ahead of
its base, and calls a diverged one a conflict ([landing.md](./landing.md#the-three-policies)). `park` needs `compare`
because finding or opening its change starts, as every policy's does, by asking whether the chunk's revision has
anything left to land.

The check runs twice. **When a policy is set** — a repository created, its policy edited, or its forge changed —
`ConfigAuthoring` checks the forge's `offered` capabilities against the policy's `requires` and refuses
`422 forge_capability_missing`, naming each missing capability, the policy, and the forge, before anything is written.
The configuration document's apply refuses the entry the same way. **When a visit resolves**, the forges epic's check
before a pass reads the action's requirements, which for `land` are no longer fixed: an action's requirements become
`requires(repository: ResolvedRepository) -> frozenset[Capability]`, declared with its `ActionSpec` in the catalogue
(`hub/domain/actions/catalogue.py`), and `land` answers its repository's policy's set. A policy whose forge stopped
offering a capability since it was set therefore escalates `forge-capability-missing` to a person, naming it, as the
forges epic escalates any missing capability. `record-findings` and `garden-deliver` still require nothing.

Some refusals are not capabilities at all, and only the landing finds them. A base protected so that only a pull request
may change it is a rule of the repository on its forge, not of the forge's kind, so setting `fast-forward` on it is
admitted, and the first advance the forge refuses goes to a person as `base-protected`
([landing.md](./landing.md#what-goes-to-a-person)).

`ResolvedRepository` carries `landing_policy`, `park_timeout_hours`, and `landing_position`, read at resolution, so a
policy edited mid-delivery takes effect at the visit's next submission and never mid-turn.

## Invariants

`tools/invariants.py` gains `hub:landing-position-on-link` — every row of `project_repository_positions` names a link of
its own tenant — and widens `hub:repository-references-in-tenant` to the table's `repository_id`.
