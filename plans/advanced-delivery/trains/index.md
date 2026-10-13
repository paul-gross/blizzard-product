# Plan — `epic:advanced-delivery`, trains slice

When many agents finish at once, landing them one by one is a slow way to discover that their work was fine. Each
landing moves the branch under every chunk still waiting. The forge brings each of those chunks up to the new base
without an agent, but every update throws away a finished run of checks and starts another, so on a busy night the line
on a branch grows faster than it drains. Most of what the fleet spends on it is re-running checks it has already passed,
and a chunk that waits through enough of those runs is turned back to an agent by its node's poll timeout, though its
work was right all along.

This slice makes the flow on a branch a train. It combines the ready work, proves the combination once, and wakes an
agent only for a conflict no machine can settle or a failure the proof pins on one chunk. On a repository that batches,
what reaches the branch is exactly the tree that was proven, so two changes that each passed alone cannot land a break
together.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

- **Mechanical first.** Inside a train, a base that moved costs a merge the forge makes, not a run of checks per chunk
  and never an agent. Only a conflict with the base itself goes back to the chunk's graph as a conflict.
- **Batches, proven once.** A flow gathers the submissions ready on its branch, combines them onto the base in the order
  they arrived, and proves the combination with one run of the repository's checks. A batch that passes lands together.
  A batch that fails is split and its halves proven apart until the chunk responsible is found, and every chunk the
  failure did not belong to lands without it.
- **The combination is what lands.** The branch advances to the very tree the flow proved, never to a tree assembled
  after the proof, so the revision that lands is the revision that passed. A lone chunk that already holds its base and
  passed its own checks is advanced to directly, under the same rule.
- **How big, how long.** A repository says how many chunks a batch may carry and how long a flow waits for a batch to
  fill before it proves what it has, so a quiet repository lands a lone chunk promptly and a busy one amortises its
  checks.
- **A train that knows when to stop.** A proof that never finishes is abandoned after the same three hours a poll may
  run, and a base whose own checks are red holds the train until it is green again, so one broken base costs one agent
  its repair rather than every chunk in line a bounce.
- **Capabilities probed per repository.** A forge's capabilities narrow to the repository: whether a merge queue or a
  merge train is in force, or whether the base accepts a direct advance, is a fact the hub reads about one repository,
  not its whole forge, and a policy or a train that needs one is checked against the repository.
- **Handing the line to the forge.** A repository whose forge runs its own merge queue — GitHub's merge queue, GitLab's
  merge trains — may hand ordering to it, as a fourth policy beside the three the `policies` slice ships. The flow
  submits each change to the forge's queue and reports the forge's verdict, so a shop that already lives on its forge's
  train keeps it. What that queue merges is the forge's promise, not the flow's.
- **Ejecting what cannot join.** A chunk whose work conflicts with the base, or whose work the split found responsible
  for a failure, is turned back to its own graph with that verdict, and the train moves on without it. The flow's record
  says which chunk was ejected, from which batch, and why.
- **The train, on the board.** A flow's lane shows the batch being proven, the chunks it carries, those ejected from it,
  and a train held for a red base, so the operator watching a busy branch sees what is moving and what is not.

## Left for later

Keeping chunks likely to touch the same ground from running side by side is not this epic's. An operator's hand on a
batch — abandoning one, or ejecting a member by choice — waits with the flows slice's levers on a branch.
