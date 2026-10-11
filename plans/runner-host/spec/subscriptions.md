# Subscriptions contract

## Declarations

`[[subscription]]` moves from the runner to the host unchanged in shape — `slug`, `name`, `provider`, optional
`credentials_path` and `sample_interval_seconds` (`runner/config.py:320`) — and the slug stays the join key the hub
already stores samples under. Each `[[runner]]` names at most one slug in `subscription`. The relation is many runners
to one subscription; a runner never spends two. A subscription is optional: it turns on tracking a plan's usage and,
under `epic:throttling`, pacing against it, and a runner working on API keys names none.

A runner's `subscription` is a declaration, not a binding: blizzard does not hand any worker the subscription's
credentials, and each harness still authenticates through the operator's own configuration of that tool. Nothing about a
runner's enabled harnesses constrains which subscription it names — a runner enabling `claude_code` and `opencode` under
one slug is valid, and a usage limit reported by either harness brakes through that one slug (§The brake).

The legacy `[external_subscription_usage]` table, and the `anthropic` declaration synthesized from it
(`resolved_subscriptions`, `config.py:779`), do not carry into `blizzard-host.toml`: a host declares its subscriptions
explicitly.

## The host store

The host keeps a store of its own, `<host dir>/data/host.db`, for what belongs to the machine rather than to a runner:

- `external_usage_samples` and `credential_renewal_claims` / `credential_renewal_outcomes`, moved from the runner store
  (`runner/store/schema.py:678`, `:696`, `:710`) with the same columns;
- `workspace_prompt`, moved from the runner store (`schema.py:377`), keyed by workspace id;
- `host_identity`, a singleton holding the host id and the tenant id and name the hub first answered with, written once
  and never cleared ([process.md](./process.md) §One tenant);
- `host_outbound_buffer`, the host's own FIFO of subscription reports awaiting delivery, shaped as a runner's
  `outbound_buffer` (`schema.py:61`).

No lease, binding, brake, or runner outbox row lives here: everything the hub's exactly-once and a runner's recovery
depend on stays in the runner's own store.

## Sampling once

`ExternalUsageSample` leaves the runner tick (`runner/loop/steps.py:690`) for a host pass on its own thread, sampling
each declared subscription on its cadence (`sample_due`, `subscriptions/subscription_sampler.py:96`) through the sampler
its provider selects (`subscriptions/internal/subscription_sampler_factory.py:24`). Credential renewal moves to the host
the same way (`CredentialRenewalPass`, `runner/composition.py:171`). A subscription no runner spends is still sampled,
and shown on the host's panel.

## Reporting to the hub

The host reports its roster in its own registration and each sample or miss once, under its own credential
(`POST /api/fleet/host/events`, [hub.md](./hub.md) §The host's fact lane). Recording a sample writes the
`external_usage_samples` row and its `host_outbound_buffer` report in one transaction, as a runner's
`record_external_usage_attempt` does today (`store/internal/usage_store.py:386`), so a sample is never recorded without
its report or reported without being recorded. A hosted runner reports no roster and no samples; it registers the slug
it spends, and the hub reads its usage from its host's.

## The brake

A usage limit is still detected per lease, by the harness the lease ran (`lifecycle/usage_limit.py:501`). What changes
is how far it reaches:

1. The runner whose worker reported the limit engages its own brake (`throttle/pause.py:310`) with `by = "usage-limit"`,
   exactly as today, and parks the lease in place.
2. If that runner spends a subscription, the host engages the same brake, with the same reason and reset time, on every
   other runner of the host spending that slug, each in its own store with its own outbound report. Runners spending
   another subscription, or none, are untouched.

The reason names the subscription beside the harness — `usage limit: anthropic via claude_code (resets …)` — so a runner
braked by a neighbour's limit says whose plan ran dry.

The reset time is the harness's own when it reported one. Otherwise `_fallback_reset` (`usage_limit.py:493`) reads the
newest sample of the runner's one subscription only, never every declared slug, and names none when that subscription
has no sample or the runner spends none.

Engagement stays once-only per brake (`LocalBrake.engage`, `pause.py:134`): a runner already braked is not re-braked by
the fan-out. Lifting is unchanged — only an operator lifts a runner's brake, one runner at a time
(`blizzard runner start --runner <id>`), and nothing lifts it at the reported reset.

A crash between the reporting runner's engagement and the fan-out leaves a neighbour unbraked. That is tolerated: the
neighbour's next worker to meet the limit engages its own brake and fans out in turn, at the cost of one parked lease.
