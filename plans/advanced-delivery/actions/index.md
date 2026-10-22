---
epic: advanced-delivery/actions
refinement: pristine
status: horizon
---

# Plan — `epic:advanced-delivery/actions`

A hub step today is a shell command. A graph's hub node lists commands under `run:`, and the hub runs each on its own
host, as its own user, with its own environment — the key that seals every tenant's secrets and the address of its store
among it. Everything blizzard ships goes through that door to run code of its own: three packaged scripts, started as
child processes and handed a token so they can call back into the hub that started them. Anyone who may author a graph,
which is every tenant's admin, may put any command there instead, and it runs with the hub's reach into every tenant.
And because each step is a process the hub starts and then waits on, the hub cannot do what the rest of this initiative
asks of it: take a landing a little at a time, beside every other, and decide for itself what comes next.

This epic makes a hub step a built-in action. A hub node names what it does — land the chunk's work, record its review
findings, deliver a garden run — and the hub does it in its own code, with nothing handed to a process. Graphs keep
every outcome they route on, every stored graph is carried over to the actions its commands already named, and `run:` is
retired.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **A hub node names an action.** A hub node says which of the hub's actions it takes, and any arguments that action
  accepts, from a set blizzard ships. A graph that names a command, or an action the hub does not have, is refused when
  it is written, and so is a hub node that leaves one of its action's outcomes without somewhere to go. A hub node that
  only routes — today an empty `run:` — takes `pass`, the action that does nothing and answers `success`.
- **The packaged scripts, as actions.** Landing a chunk's work once its checks pass, recording a review's deferred
  findings, and delivering a garden run become the hub's `land`, `record-findings`, and `garden-deliver` actions. Each
  answers with the outcomes and writes the markers its script did, so every shipped graph routes as it does today.
  Landing uses the repository's credential inside the hub, and the credential never leaves it.
- **An action works only with what it is handed.** An action is given everything its visit needs before it starts — the
  chunk's repositories, its artifacts, the markers already written at its node, and a connection to each forge it lands
  on — and it reaches nothing else: not the store, not a secret, not the hub's configuration. What it records, it hands
  back to the hub, which makes each record durable before the action goes on. An action that needs more declares it as
  an input and is handed it; the rule is enforced by the hub's own tests, so a later change that reaches past it fails
  the build rather than the night. Actions run on a thread of their own, so a slow forge never slows the board's pages
  or its event stream; until the `flows` epic, the runner's call that reached a hub node still waits on its pass, as it
  waits on a script today.
- **An action takes a pass, and says what comes next.** Each time the hub calls an action, the action does a bounded
  piece of work and answers with an outcome or with `pending`. A pass is cut off at five minutes by default, a limit an
  operator can change in the hub's configuration; a landing that waits on checks is a run of passes, never a process the
  hub waits on, which is the shape the `flows` epic drives.
- **A merge, once made, is on the record.** A chunk stopped, restarted, or moved on while its landing merges still has
  that merge recorded, since the branch carries the work whatever became of the chunk, and the landing stops there
  rather than merging the chunk's next repository on its behalf.
- **Patience in hours, not minutes.** A hub node that keeps answering `pending` gives up and goes back to an agent only
  after three hours by default, not thirty minutes, so a busy branch's CI no longer sends correct work back for repair.
  A graph still sets its own limit per node.
- **Carried over in place.** Every stored graph, in every tenant and at every version, that names a packaged script
  takes the action that script was, and nothing is re-pinned. A version that landed through the old unchecked script now
  waits on checks like every other. A command that names anything else, and none is expected, no longer runs: a chunk
  that reaches it, or already stands at it, goes to a person naming the command, and so does a chunk whose action
  answers an outcome its graph never routed.
- **The door closed.** The hub stops starting processes for graphs, and the environment and per-visit token it handed a
  step are removed, so nothing a tenant writes runs with the hub's reach.
- **Retired the way anything released is.** What only the scripts ever called — the routes the token unlocked,
  `blizzard hub record-marker`, the scripts' working directories on disk, and the step's trace span — is announced as
  deprecated in the release that stops running `run:`, and removed in a later release of its own, so no operator meets a
  removal no release warned them of.

## Left for later

Delivery logic a tenant writes for itself, run where the tenant already trusts its own code — a command step on its own
runners, or a service of its own that the flow's landed event calls — waits for a tenant who needs one.
