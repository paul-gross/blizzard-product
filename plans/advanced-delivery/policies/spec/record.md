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
`StrEnum` in `hub/domain/config/landing_policies.py`, beside `POLICY_CAPABILITIES`, what each policy requires of the
forge and what it uses when offered ([Capabilities](#capabilities)), `DEFAULT_PARK_TIMEOUT_HOURS`, and `PARK_POLL_FLOOR`
([landing.md](./landing.md#awaiting-a-merge)). The module sits in the domain's `config` package, which imports no other
domain package, so the repository record, `ConfigAuthoring`'s check, the actions catalogue, and the forge adapters all
read the same table without a layer crossing.

`validate_fields` refuses an unknown policy and a `park_timeout_hours` below one. A timeout belongs only to a parked
repository, so an edit that moves a repository off `park` clears its `park_timeout_hours` in the same revision —
`RepositoryEdit.merge` drops it, and the configuration document's apply does the same — rather than refusing the edit; a
write that sets a timeout on a repository whose resulting policy is not `park` is refused.

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
  INDEX (tenant_id, project_id, repository_id, id)
```

The index serves the newest fact per link, the read `project_repository_link_facts` is shaped for.

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

## When a visit takes its terms

An operator who changes a repository's policy, or reorders a project, wants it to take hold now. A chunk can wait in a
branch's line for hours before the hub first acts on it, and binding it to the policy it saw when it joined that line
would carry a stale choice into the landing; binding it afresh on every pass would let a landing change its nature
halfway, opening a pull request to merge it and then leaving it parked. So a visit takes its terms at the moment its
delivery starts, and keeps them.

A visit's **landing terms** are, for each repository it holds, that repository's `landing_policy` and
`park_timeout_hours` and its position in the chunk's project's order. They are written once, by the transaction that
starts the submission's first turn ([flows' turns](../../flows/spec/turns.md#a-turn)), into a table beside the flows
epic's submissions:

```text
flow_submission_terms                    the terms a visit lands by, written at its first turn
  submission_id     String   NOT NULL   FK flow_submissions
  tenant_id         String   NOT NULL   FK tenants
  repository_id     String   NOT NULL
  landing_policy    String   NOT NULL
  park_timeout_hours Integer NULL
  landing_position  Float    NULL       null is unranked
  PRIMARY KEY (submission_id, repository_id)
  INDEX (tenant_id, submission_id)
```

`submission_id` is marked `hub_unique`, as the flows epic marks its ids for the tenant schema gate. The transaction that
starts a turn reads the held repositories' policies and park timeouts and the project's positions, and writes them only
where the submission has none, before it commits and before the visit is built, so a turn retried after an interruption
keeps the terms its first attempt took. Every later pass of the visit reads its terms from the submission, never from
the records. An edit made while the chunk waits in line before that first turn is the one it lands by; an edit after it
waits for the chunk's next visit, which a requeue, a restart, or a migration onto the node starts, at the back of its
lines, with fresh terms. What the flows epic re-reads every pass — the base branch, the forge, the credential — it still
re-reads; the terms are only the three fields above.

The visit carries its terms to each `RepoLanding` ([landing.md](./landing.md#land-runs-policies)). `LandingProject`, the
project a visit lands for, moved into `hub/domain/actions/visit.py` with the actions epic; it gains `positions`, each
linked repository's current position or `None`, read by the driver for the first turn's terms. `ResolvedRepository`
gains `landing_policy`, which the driver stamps from the visit's terms once its first turn has taken them, and from the
repository record before then, so the catalogue's `requires(repository)` reads the policy the visit lands by without
reaching for the terms itself.

## Capabilities

Each policy names the capabilities it cannot do without, and those it uses when offered, in `POLICY_CAPABILITIES`:

| Policy            | Requires                                                                                                                 | Uses when offered                              |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------- |
| `fast-forward`    | `refs.read`, `compare`, `refs.advance`, `checks.read`                                                                    | `branches.merge`, `refs.create`, `refs.delete` |
| `merge-on-checks` | `refs.read`, `compare`, `changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, `checks.read` | `checks.rerun`                                 |
| `park`            | `refs.read`, `compare`, `changes.open`, `changes.read`                                                                   | —                                              |

`refs.advance`, `branches.merge`, `refs.create`, and `refs.delete` are new to the closed `Capability` set
(`hub/domain/config/capabilities.py`, [seam.md](./seam.md#the-seam-grows)). GitHub offers all four; a forge kind that
offers none cannot fast-forward, and says so when an operator tries. `fast-forward` without the three it uses still
lands a revision that is already ahead of its base, and calls a diverged one a conflict
([landing.md](./landing.md#the-three-policies)). `park` needs `compare` because finding or opening its change starts, as
every policy's does, by asking whether the chunk's revision has anything left to land.

The check runs twice. **When a policy is set** — a repository created, its policy edited, or its forge changed —
`ConfigAuthoring` asks an `IForgeOffers` port (`hub/domain/config/capabilities.py`), which composition answers with
`ForgeRegistry.offered` — the one place a caller asks, as the forges epic made it, answering the kind's set in this epic
and a repository's probed set once the `trains` epic narrows it; for a repository not yet created it answers its forge's
kind — and checks the answer against the policy's requirements and refuses `422 forge_capability_missing`, naming each
missing capability, the policy, and the forge, before anything is written. The configuration document's apply refuses
the entry the same way. **When a visit resolves**, the forges epic's check before a pass reads the catalogue's
`ActionSpec.requires(repository)` (`hub/domain/actions/catalogue.py`), which the forges epic declared as a function of
the repository from the start, answering a fixed set for `land`. This epic changes only `land`'s function: it answers
`POLICY_CAPABILITIES[repository.landing_policy].requires`, the policy the driver stamped from the visit's terms. A
policy whose forge stopped offering a capability since it was set therefore escalates `forge-capability-missing` to a
person, naming it, as the forges epic escalates any missing capability. `record-findings`, `garden-deliver`, and `pass`
still require nothing.

Some refusals are not capabilities at all, and only the landing finds them. A base protected so that only a pull request
may change it is a rule of the repository on its forge, not of the forge's kind, so setting `fast-forward` on it is
admitted, and the first advance the forge refuses goes to a person as `base-protected`
([landing.md](./landing.md#what-goes-to-a-person)).

## Invariants

`tools/invariants.py` gains `hub:landing-position-on-link` — every row of `project_repository_positions` names a link of
its own tenant — and `hub:submission-terms-standing` — every `flow_submission_terms` row names a repository its
submission holds a flow for, and every submission with an open turn has a row for each repository it holds — open rather
than started, since a submission in flight at the upgrade has turns from before the terms existed and takes its terms at
its next turn, every turn open at the upgrade having been ended `interrupted` at boot — and widens
`hub:repository-references-in-tenant` to both tables' `repository_id`.
