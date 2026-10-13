# GitLab specifications

The technical contracts behind the gitlab slice of `epic:advanced-delivery`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. The forge record, the seam's roles, its
vocabulary, and its retry rules are owned by the [forges slice](../../forges/spec/index.md); the policies, the seam
operations they added, the order a project lands its repositories in, the combining step, and the rate budget by the
[policies slice](../../policies/spec/index.md). This spec adds a second adapter behind that seam and changes no policy.

This spec assumes the [actions](../../actions/spec/index.md), [forges](../../forges/spec/index.md),
[flows](../../flows/spec/index.md), and [policies](../../policies/spec/index.md) slices have landed: `land` runs each
repository's policy a pass at a time, a pure action under `bzh:hub-action-purity`; every forge is spoken through the
async roles `ForgeRegistry` hands out — `IForgeRefs`, `IForgeChanges`, and `IForgeChecks`, with the operations the
policies slice added ([seam.md](../../policies/spec/seam.md)); and `ForgeBudgets` keys each forge's rate budget by
forge, secret, and bucket ([budget.md](../../policies/spec/budget.md)).

Paths are relative to `blizzard/src/blizzard/`.

The slice reaches beyond `blizzard`'s source. In `blizzard-context`, `domain/artifacts/delivery.md` names GitLab beside
GitHub as a forge a repository may land on; `verification/blizzard/e2e-scenarios/delivery-policy.md` gains the GitLab
scenarios; and blizzard's verifiability matrix declares the GitLab contract suite as its own method
([carry-over.md](./carry-over.md#the-contract-suite)). `docs/deployment/forges.md` describes declaring a GitLab forge,
the minimum version, and the policies a GitLab repository may choose. `blizzard-mock`'s forge gains a GitLab face
([carry-over.md](./carry-over.md#the-mock)).

| Where                            | Read when                                                                                                          |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [adapter.md](./adapter.md)       | Building the GitLab adapter — its operations, capabilities, vocabulary, failures, and what its rate headers teach  |
| [landing.md](./landing.md)       | Running the policies on GitLab — the head gate for a rebased change, what each policy proves, landing across kinds |
| [namespaces.md](./namespaces.md) | Letting a repository live in nested groups — the owner, `RepoRef`, and GitLab's web addresses                      |
| [carry-over.md](./carry-over.md) | Shipping the slice — the runner and hub together, the mock's GitLab face, the contract suite, and the proof        |
