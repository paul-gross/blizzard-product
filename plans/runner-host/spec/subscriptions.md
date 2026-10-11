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

## Where samples live

Samples, misses, and renewal records are host-scoped rows of the host store — `external_usage_samples`,
`credential_renewal_claims`, `credential_renewal_outcomes` (`runner/store/schema.py:678`, `:696`, `:710`), with the same
columns ([store.md](./store.md) §Runner-scoped and host-scoped tables). A sample belongs to its slug, not to any runner
that spends it.

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
its report or reported without being recorded. A runner of a registered host reports no roster and no samples; it
registers the slug it spends, and the hub reads its usage from its host's.

An unregistered host has no lane of its own ([configuration.md](./configuration.md) §Hub credentials). Its one runner
reports its roster and each sample through its own outbox, exactly as a runner does today, and the host writes the
sample and that runner's `outbound_buffer` report in one transaction.

## The brake

A usage limit is still detected per lease, by the harness the lease ran (`lifecycle/usage_limit.py`). What changes is
how far it reaches:

1. The runner whose worker reported the limit engages its own brake (`throttle/pause.py:310`) with `by = "usage-limit"`,
   exactly as today, and parks the lease in place.
2. If that runner spends a subscription, the same transaction engages the same brake, with the same reason and reset
   time, on every other runner of the host spending that slug, each with its own `local_pause_facts` row and its own
   outbound report. Runners spending another subscription, or none, are untouched.

Because every runner's brake lives in the one host store, the fan-out is a single `BEGIN IMMEDIATE` transaction with the
reporting runner's own engagement: either every runner spending the plan is braked, or none is and the limited lease is
not yet parked.

The reason names the subscription beside the harness — `usage limit: anthropic via claude_code (resets …)` — so a runner
braked by a neighbour's limit says whose plan ran dry.

The reset time is the harness's own when it reported one. Otherwise `_fallback_reset` (`usage_limit.py:137`) reads the
newest sample of the runner's one subscription only, never every declared slug, and names none when that subscription
has no sample or the runner spends none.

Engagement stays once-only per brake (`LocalBrake.engage`, `pause.py:134`): a runner already braked is not re-braked by
the fan-out. Lifting is unchanged — only an operator lifts a runner's brake, one runner at a time
(`blizzard runner start --runner <id>`), and nothing lifts it at the reported reset.
