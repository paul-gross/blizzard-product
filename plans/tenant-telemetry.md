---
epic: tenant-telemetry
refinement: scaffolded
slices:
  - name: full
    status: horizon
---

# Plan — `epic:tenant-telemetry`

A hub that hosts several tenants narrates every one of them to a single place. Its traces go to the observability
backend whoever runs the hub chose, each span tagged with the tenant it belongs to, and that suits the person who runs
the hub. It does not suit the tenant. The team whose fleet lives in one tenant already has an observability backend of
its own, and the story of its nights arrives somewhere it cannot see, mixed with other tenants' stories it should never
be able to read.

This epic lets each tenant name where its own telemetry goes. The hub keeps its hub-wide export for the person who
operates it, and beside it forwards each tenant's traces — and, where `epic:fact-egress` writes its files, each tenant's
facts — to the destination that tenant declared, with that tenant's credentials, and nothing of any other tenant's. A
tenant that declares nothing is narrated only to the hub's own destination, as today.

## What to build

- **A destination per tenant.** An OpenTelemetry endpoint, its protocol and headers, and a credential from the tenant's
  secret store, configured the way `epic:live-config` configures every other record.
- **Forwarding without leaking.** Every span, and every egress file, reaches only the tenant it belongs to; the tenant
  attribute that tags hub-wide export today is what routes it.
- **Only a tenant's own story.** A tenant receives the spans of its own requests and chunks and the fact rows that name
  it. The hub's sweep spans are the hub's own: they carry no tenant, and no tenant ever receives them.
- **A backend that is slow or down costs only its own tenant.** One tenant's unreachable collector never delays another
  tenant's export or the hub's own.

## What it stands on

`epic:multi-tenancy` names the tenant on every request span and every exported fact row, and keeps trace export and fact
egress hub-wide; this epic is the step past it. The hub's sweep spans stay tenant-less. The fleet-spans, platform-spans,
and runner-spans slices of `epic:tracing` define what a span is.

## Open questions

- Whether runners, which export their own spans locally, forward per tenant too, or only the hub's spans are routed.
- Whether a tenant's destination receives the platform spans of the requests it made, or only the spans of its own
  chunks. Sweep spans are settled: they never go to a tenant.
- Whether per-tenant fact egress belongs here or in `epic:fact-egress`.
