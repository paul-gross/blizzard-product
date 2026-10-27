---
epic: advanced-forge-auth
refinement: scaffolded
status: horizon
---

# Plan — `epic:advanced-forge-auth`

An operator who gives the fleet a private repository needs to know that it can finish the work it accepts. Opening a
pull request is not enough: the credential must also let the hub read its checks and merge it when the repository
permits. Today an incompatible credential can survive setup and fail only after an agent has finished, with an
explanation that says little more than “authentication failed.” For `persona:application-architect`, that is a night of
unattended work returned as a configuration problem they could have settled before it began.

This epic gives a tenant an explicit choice of forge authentication, with GitHub PATs and operator-owned GitHub Apps as
its first bindings. The hub distinguishes a token the operator maintains from credentials through which it obtains
short-lived access itself. Repositories and work sources select their authentication independently; the rest of blizzard
asks its forge to do the work without knowing how it authenticated.

## The boundary

A forge identifies a server and its kind. A credential identifies how a particular use authenticates to that server. A
tenant may use several credentials on one forge, and several repositories and work sources may share one credential.
Reading issues need not carry the same privileges as landing changes. Neither a credential nor the ability to use it
crosses the tenant boundary.

The useful common distinction is who maintains access, not which word appears in an HTTP header. With an **externally
supplied token**, the operator supplies and replaces the secret; blizzard does not invent a way to renew it. With
**provider-managed access**, the operator supplies an identity and its authorization, and blizzard obtains and renews
the access tokens that authorization permits. A GitHub App installation is one such authorization. An OAuth grant is
another, with a different setup and renewal protocol; GitHub App installation authentication is not OAuth, and a PAT may
travel in the same bearer header as a managed token.

The core owns credential references, lifecycle, tenant isolation, and a narrow boundary for obtaining usable
authentication. Each provider owns its configuration variants, permission vocabulary, token exchange, and request
authentication. GitLab and other forges must fit without teaching delivery policies their protocols, but this epic
builds no speculative universal OAuth engine and requires no GitHub fields of another provider.

## What to build

- **Authentication declared once and selected per use.** A tenant manages typed credential records through the API, CLI,
  configuration document, and board, with the revision, change history, retirement, and secret handling other configured
  records have. Repositories and forge-backed work sources reference a credential compatible with their forge. A record
  exposes its kind and non-secret setup, never a token or private key. Replacing key material or changing a binding
  reaches its next use without a hub restart.
- **PATs remain a straightforward option.** A supplied token uses the existing secret store. Existing repository and
  work-source secret references keep working through an additive transition, without an operator re-entering a token.
  GitHub setup distinguishes classic and fine-grained PATs, documents their supported workflows and permissions, and
  refuses known unsupported combinations rather than treating every PAT as interchangeable. A classic PAT remains a
  supported private-repository path; fine-grained PATs are not promised Checks API access GitHub does not support.
- **Operator-owned Apps, from setup through renewal.** An administrator creates an App on GitHub, installs it on the
  chosen account or organization and repositories, and registers its App identity, installation, and private-key secret
  in their tenant. One App may serve many repositories; an App is not required per project or user. The hub obtains
  installation tokens, narrows their access where appropriate, and renews them before expiry, coordinating concurrent
  renewal and recovering after a restart. Expired access, revoked or suspended installations, removed repositories,
  changed permissions, and key replacement receive explicit handling. Cached authorization never mixes tenants, forge
  servers, installations, or access scopes.
- **Every hub caller uses the same authentication boundary.** Delivery, commit verification, issue ingestion, and
  work-source annotations and closes use the selected credential through their existing provider adapters. The provider
  handles token acquisition and request authentication; actions, flows, and landing policies receive no credential
  material. A long-lived client cannot pin an installation token past its usable lifetime. Write coordination and rate
  accounting follow the credential's effective authorization rather than treating each renewed token as an unrelated
  identity.
- **Readiness checked before expensive work.** A read-only integration check reports repository access and known
  permission or token-kind gaps for the operations that use requires. It exercises the relevant reads, including
  private-repository checks, rather than accepting a successful repository read as proof of delivery access. Unknown or
  unprovable write permissions are stated honestly: validation never merges, changes a branch, or opens a probe pull
  request, and cannot guarantee that a future change satisfies repository policy. Known incompatibilities are surfaced
  before work is promoted into the affected delivery path.
- **Refusals an operator can act on.** Failed operations retain a sanitized provider message, HTTP status, operation or
  endpoint, repository, and request identifier when available. Invalid or expired credentials, insufficient access,
  unsupported token capabilities, and repository-policy refusals are distinguishable from rate limits and outages.
  Tokens, keys, and authorization headers never enter diagnostics. Unavailable, unreadable, or unexpectedly absent
  checks never become permission to merge. An operator can replace the credential, check it, and recover an
  already-escalated delivery without duplicating its pull request or losing recorded landings.

## Contained slices

Keep this one epic with three substantial implementation slices, not a separate child for every surface or adapter. Each
slice includes its own tests, operator documentation, and compatible upgrade path.

| Slice                                                                 | Outcome                                                                                                                                                                                                                                                                                                                                                  |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Explicit credentials, with supplied tokens working end to end.** | Establish the provider-aware credential boundary and tenant-owned records, wire repositories and work sources through it, carry existing PAT setup forward, and expose the complete configuration surface. Preserve actionable provider refusals as part of the port. The fleet still ingests and lands through supported PATs at the end of this slice. |
| **2. GitHub App installations working end to end.**                   | First prove the exact private-repository Checks read with a real installation token, then deliver App setup, acquisition and renewal across every hub caller, including expiry, concurrent refresh, revocation, and key replacement. Setup and recovery instructions ship with the implementation, not as a documentation-only follow-up.                |
| **3. Readiness and recovery before unattended use.**                  | Deliver the read-only validation surface, known-capability checks before promotion into delivery, and the operator's check–repair–resume loop. Prove that insufficient permissions and lost access remain safe throughout an in-flight delivery and that each refusal names what the operator must change.                                               |

The real-provider proof at the start of slice 2 is a gate, not another platform feature: install a test App on an
operator-approved private repository, use its installation token to read a PR's head and the exact check-runs endpoint
delivery uses, and verify the expected checks and conclusions rather than merely a successful HTTP status. It creates no
probe PR and performs no merge. Keep credentials and private response content out of committed evidence. Automated tests
against the mock forge then cover the lifecycle and refusal cases reproducibly; a public-repository success is not a
substitute for the private-repository proof.

## Relationship to delivery and compatibility

This epic builds on the forge record and landing seam of `epic:advanced-delivery/forges`, and coordinates its
configuration changes with that epic's forge surfaces. It does not wait for branch flows, landing policies, GitLab, or
merge trains. Those consumers use authenticated provider operations; none owns a token-renewal protocol. Once
implementation is filed, its prerequisites must name the actual issues or landings it builds on rather than declaring
the whole advanced-delivery initiative a blocker.

Every landing is additive to supported releases. Existing PAT secrets, references, configuration documents, and clients
remain usable while the new credential configuration arrives. A retirement of superseded secret-reference fields, if
needed, is a separate follow-up cleanup outside this epic, after the transition has shipped. Authentication here belongs
to hub-to-forge integrations; runner checkout and worker-side git credentials remain their own boundary.

## Deliberately left out

- A GitLab or other forge authentication binding. The core admits one without a delivery rewrite; the provider's own
  integration supplies it when needed.
- An implemented OAuth authorization flow or generic grant framework without a concrete provider use case. Its eventual
  binding may reuse the managed-access lifecycle, not pretend that a GitHub installation is an OAuth grant.
- A Blizzard-owned public App shared across hosted customers, a central credential broker for independent hubs, or
  distributing a shared App private key. Operator-owned Apps are sufficient for this epic.
- Creating Apps from inside blizzard, installation onboarding through GitHub manifests, and a marketplace listing.
  Administrators create and install their App on GitHub.
- Webhook registration, inbound event handling, and replacing polling. App authentication makes those possible but works
  without an inbound hub endpoint.
- Changing GitHub login, implementing CI/CD, or granting an App a bypass around branch protection and approval policy.

## Questions for refinement

- The smallest provider-aware credential schema that represents supplied tokens and App installations, permits key reuse
  across installations, and keeps the operator's setup understandable. Decide whether App registrations need records of
  their own only if their lifecycle earns that extra concept.
- How existing direct secret references map onto credential records without ambiguity when ingestion and delivery share
  a secret, and how the transition stays readable to supported clients.
- Which readiness checks can establish each required permission without writes, which results are uncertain, and how
  their freshness is invalidated by credential, repository, or policy changes. Permission labels alone must not stand in
  for a successful supported read.
- The precise recovery operation for a delivery already escalated on authentication, preserving its existing PR and
  landed facts, and how installation-scoped authorization fits the shared forge rate budget.

## Provider references

- [GitHub App installation authentication](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation)
- [GitHub App permissions for Checks](https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps#repository-permissions-for-checks)
- [GitHub PAT limitations](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens-limitations)
