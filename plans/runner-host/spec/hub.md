# Hub contract

## The host record

A host is a tenant-scoped record beside the runner registry (`runner_registrations`, `hub/store/schema.py:1750`):

| Column          | Meaning                                                                                             |
| --------------- | --------------------------------------------------------------------------------------------------- |
| `host_id`       | `hs_` plus a ULID, minted by the hub, never reused                                                  |
| `tenant_id`     | The one tenant the host serves, fixed at creation                                                   |
| `name`          | The operator's label, unique among the tenant's live hosts                                          |
| `redirect_uris` | The panel's sign-in callbacks, reported by the host                                                 |
| `url`           | The panel's canonical origin, reported by the host                                                  |
| `workspaces`    | JSON list of `{workspace_id, env_capacity}`, reported by the host and replaced whole on each report |
| `subscriptions` | JSON roster of `{slug, name, provider}`, reported by the host and replaced whole on each report     |
| `registered_at` | First report; null while the host has never connected                                               |
| `last_seen_at`  | Newest report                                                                                       |

Retirement is a lifecycle fact, as a runner's is (`runner_lifecycle_facts`, `schema.py:1808`). `runner_registrations`
gains `host_id NULL`: set when a runner is added under a host or adopted into one, and null for a runner that has no
host.

## The host credential

A host holds one bearer token, stored at the hub as a hash and revocable on its own, the way a runner's token is
(`runner_token_revocations`, `schema.py:1823`). It authenticates a new principal, the **host principal**, resolved by
`presented_bearer` (`hub/api/bearer.py:13`) alongside the runner principal. The host principal is bound to its host's
tenant (`tenant_bound`) and may do only the following:

| Action                               | Route                                             |
| ------------------------------------ | ------------------------------------------------- |
| Report itself                        | `POST /api/fleet/host`                            |
| Report a subscription sample or miss | `POST /api/fleet/host/subscription-usage`         |
| Add a runner under itself            | `POST /api/fleet/host/runners`                    |
| Retire a runner under itself         | `POST /api/fleet/host/runners/{runner_id}/retire` |
| Read its own record and its runners  | `GET /api/fleet/host`                             |

Every other `/api/fleet` route stays behind `require_runner_principal` (`hub/api/fleet.py:118`), so a host token cannot
peek, claim, report a lease, or register as a runner. A runner token cannot act as its host.

Adding a runner under the host mints a runner id and token exactly as `POST /api/runners` does
(`hub/api/runners.py:257`), in the host's tenant, with `host_id` set and the runner's initial project declaration taken
from the request. The token is returned once, in the response. Retiring a runner under the host follows
`POST /api/runners/{runner_id}/retire` (`runners.py:344`), including its refusal while the runner holds live chunks
unless forced.

## Operator verbs

Under an operator session, with `RUNNER_RETIRE`-level permission for the destructive ones:

- `hub host add <name> [--tenant <id>] [--adopt <runner id>…]` creates the host and prints its id and token once.
  `--adopt` places existing runners under it, refusing a runner in another tenant, one already under a host, or one
  retired.
- `hub host list`, `hub host show <id>`.
- `hub host adopt <id> <runner id>…` places existing runners under an existing host, by the same rules.
- `hub host revoke-token <id>` revokes the host's credential and issues none; `hub host enroll <id>` issues a new one,
  as `POST /api/runners/{runner_id}/enrollments` does for a runner (`runners.py:280`).
- `hub host retire <id> [--force]` revokes the credential and retires every runner under it, refusing as a runner's
  retirement does while any of them holds live chunks.
- `hub runner add <name> --host <id>` adds a runner under a host from an operator's session.

## Registration

The host reports itself on every host tick through `POST /api/fleet/host`: `name`, `url`, `redirect_uris`, `workspaces`,
and `subscriptions`. The answer carries the host's id, tenant id and name, and the runners the hub holds under it.

A runner's registration (`RunnerRegistrationRequest`, `wire/runner.py:53`) gains, additively under
`bzh:fleet-wire-additive`:

```python
bindings: list[RunnerBinding] | None = None   # {project, workspace_id, agents}
subscription: str | None = None                # the slug it spends, one of its host's roster
```

A runner under a host sends `bindings`, from which the hub derives the served-projects declaration exactly as it does
from `projects` today ([eligibility](../../../delivered/projects/hub/spec/eligibility.md)), and sends neither `projects`
nor a subscription roster. Its `workspace_id` and `env_capacity` become optional; a hosted runner omits them, since its
host reports its workspaces. A runner with no host keeps sending what it sends today.

Registration doubles as the heartbeat, so it is never refused over its content. A binding naming a workspace its host
has not reported, or a subscription outside the host's roster, is stored and flagged on the board and in the answer, as
an unresolved project entry is today, and the hub offers the runner no work through that binding.

## Sign-in

The host is the panel's sign-in client. `authorize` (`hub/api/idp.py:183`) accepts a host id as `client`, checks the
`redirect_uri` against the host's reported `redirect_uris`, and mints a token whose audience is the host id. The role it
carries is the person's role in the host's tenant, as for a runner today. A runner with no host keeps signing in as its
own client.

## Subscription usage

A host's samples and misses are stored against the host and slug, in tables shaped as `runner_external_usage` and
`runner_external_usage_misses` are (`schema.py:1865`, `:1887`). A hosted runner's view (`RunnerView.subscriptions`,
`wire/runner.py:163`) shows its host's newest sample for the slug it spends, so every reader of a runner's usage keeps
working; the host's view shows each sample once, with the runners spending it named. A runner with no host keeps
reporting its own roster and samples.

## The board

The fleet view groups runners under their host: a host card holding its runners, its workspaces with their environments
in use, and its subscriptions with the runners spending each. A runner with no host stands on its own, as today.
Operator controls stay per runner — pausing, resuming, and retiring a runner are unchanged — and the host card adds only
the host's own retirement and credential verbs.
