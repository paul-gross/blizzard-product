---
epic: terraform-provider
refinement: scaffolded
slices:
  - name: full
    status: horizon
---

# Plan — `epic:terraform-provider`

A shop that adopts blizzard already describes its infrastructure somewhere, and for most shops that somewhere is
Terraform. The machine the hub runs on, its DNS, its backups — each is a resource in a plan someone reviews before it is
applied. Then the hub comes up, and everything that makes it *this* shop's hub — its tenants and who belongs to them,
its projects, the sources each project reads and the repositories it lands into, the credentials they use — is
configured with a different tool, in a different file, and reviewed in nobody's plan. Our own hosted hub lives exactly
this split: its host is Terraform in its infrastructure repository, and its contents are not.

This epic lets [`persona:application-architect`](../charter/personas/application-architect.md) declare a hub's
configuration where the rest of the estate already lives. A Terraform provider for blizzard manages the hub's configured
records as resources, so the plan that creates the host also states what runs on it, and a change to a project's
repositories is reviewed and applied like a change to a DNS record.

## What to build

- **A resource for every configured record.** Tenants and memberships, projects, work sources, repositories, and
  secrets, each a resource with a data source to read it; landing policies and deployment declarations join them as
  `epic:advanced-delivery` and `epic:advanced-deployment` give them a home.
- **Destroy means retire.** Removing a resource from a plan retires the record rather than erasing it, so a source's
  history stays with the chunks that came from it and a mistaken destroy can be undone from the hub.
- **Secrets that never reach state.** A secret is write-only: the provider sends it and the hub never returns it, so
  neither a plan nor a state file ever carries a credential.
- **Published where operators look.** The provider is released to the public Terraform registry, works with OpenTofu,
  and is versioned against the hub's API, so an operator can tell which provider a given hub accepts.

## What it stands on

The provider is a client of the hub's configuration API and nothing more. `epic:live-config` builds that API with this
provider named as its second consumer, and the guarantees it makes — stable identifiers, idempotent writes, a difference
shown before anything is written, retirement in place of deletion, and secrets that are written but never read — are
what keep the provider thin. It cannot usefully start until `epic:multi-tenancy` and `epic:projects` have given the hub
the records worth managing; built any earlier, it would be rewritten as each of them lands.

## Open questions

- Whether the provider lives in blizzard's own repository or one of its own — the registry expects a repository per
  provider, and the provider is written in Go where blizzard is not.
- What an apply does when it would retire a record that still carries live work: refuse, or retire it and let the work
  already in flight finish.
- Whether the provider reaches hub-level records — enrolled runners, the administrator's tenant list — or stops at what
  a tenant owns.
- Whether `epic:live-config`'s declarative file and this provider both remain, or one becomes the recommended path for
  configuration kept in a repository.
