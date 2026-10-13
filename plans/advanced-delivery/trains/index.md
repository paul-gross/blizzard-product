# Plan — `epic:advanced-delivery`, trains slice

When many agents finish at once, landing them one by one is a slow way to discover that their work was fine. Each
landing moves the branch under every chunk still waiting, the next chunk's checks are stale before they finish, and a
chunk whose work was already right goes back to an agent to be brought up to a base that has moved twice since. On a
busy night the line on a branch grows faster than it drains, and most of what the fleet spends on it is waiting and
re-running, not building.

This slice makes the flow on a branch a train. It does the mechanical work itself — bringing a change up to its base,
combining ready work, proving the combination once — and wakes an agent only for a conflict no machine can settle. What
reaches the branch is exactly the tree that was proven, so two changes that each passed alone cannot land a break
together.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

- **Mechanical first.** A change behind its base is brought up to it by the flow, through its forge, and only a textual
  conflict the forge cannot bring it past goes back to the chunk's graph as a conflict. Nothing an agent did is redone
  because the branch moved.
- **Batches, proven once.** A flow gathers the submissions ready on its branch, combines them onto the base in the order
  they arrived, and proves the combination with the repository's checks. A batch that passes lands together. A batch
  that fails is split and its halves proven apart until the chunk responsible is found, and every chunk the failure did
  not belong to lands without it.
- **The combination is what lands.** The branch advances to the very tree the flow proved, never to a tree assembled
  after the proof, so the revision that lands is the revision that passed.
- **How big, how long.** A repository says how many chunks a batch may carry and how long a flow waits for a batch to
  fill before it proves what it has, so a quiet repository lands a lone chunk promptly and a busy one amortises its
  checks.
- **Capabilities a repository switches on.** A forge's capabilities narrow to the repository: whether a merge queue or a
  merge train is enabled is a fact about one repository, not its whole forge, and a policy or a train that needs one is
  checked against the repository.
- **Handing the line to the forge.** A repository whose forge runs its own merge queue — GitHub's merge queue, GitLab's
  merge trains — may hand ordering to it, as a fourth policy beside the three the `policies` slice ships. The flow
  submits each change to the forge's queue and reports the forge's verdict, so a shop that already lives on its forge's
  train keeps it.
- **Ejecting what cannot join.** A chunk whose work conflicts with the batch, or whose work the split found responsible
  for a failure, is turned back to its own graph with that verdict, and the train moves on without it. The flow's record
  says which chunk was ejected, from which batch, and why.
- **The train, on the board.** A flow's lane shows the batch being proven, the chunks it carries, and those ejected from
  it, so the operator watching a busy branch sees what is moving and what is not.

## Left for later

Building speculatively along the line — proving each queued chunk on top of everything ahead of it at once, so a long
line drains in the time of one proof — waits until batching alone is shown too slow. Keeping chunks likely to touch the
same ground from running side by side is not this epic's.
