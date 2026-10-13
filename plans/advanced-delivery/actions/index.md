# Plan — `epic:advanced-delivery`, actions slice

A hub step today is a shell command. A graph's hub node lists commands under `run:`, and the hub runs each on its own
host, as its own user, with its own environment — the key that seals every tenant's secrets and the address of its store
among it. Everything blizzard ships goes through that door to run code of its own: three packaged scripts, started as
child processes and handed a token so they can call back into the hub that started them. Anyone who may author a graph,
which is every tenant's admin, may put any command there instead, and it runs with the hub's reach into every tenant.
And because each step is a process the hub starts and then waits on, the hub cannot do what the rest of this epic asks
of it: take a landing a little at a time, beside every other, and decide for itself what comes next.

This slice makes a hub step a built-in action. A hub node names what it does — land the chunk's work, record its review
findings, deliver a garden run — and the hub does it in its own code, with nothing handed to a process. Graphs keep
every outcome they route on, every stored graph is carried over to the actions its commands already named, and `run:` is
retired.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

- **A hub node names an action.** A hub node says which of the hub's actions it takes, and any arguments that action
  accepts, from a set blizzard ships. A graph that names a command, or an action the hub does not have, is refused when
  it is written.
- **The packaged scripts, as actions.** Landing a chunk's work once its checks pass, recording a review's deferred
  findings, and delivering a garden run become the hub's `land`, `record-findings`, and `garden-deliver` actions. Each
  answers with the outcomes and writes the markers its script did, so every shipped graph routes as it does today.
  Landing uses the repository's credential inside the hub, and the credential never leaves it.
- **An action takes a pass, and says what comes next.** Each time the hub calls an action, the action does a short,
  bounded piece of work — a handful of calls to a forge at most — and answers with an outcome or with `pending`. A
  landing that waits on checks is a run of short passes, never a process the hub waits on for minutes, which is the
  shape the `flows` slice drives.
- **Carried over in place.** Every stored graph, in every tenant and at every version, that names a packaged script
  takes the action that script was. A chunk pinned to an old version lands exactly as it did, and nothing is re-pinned.
  A command that names anything else, and none is expected, no longer runs: a chunk that reaches it goes to a person,
  naming the command.
- **The door closed.** The hub stops starting processes for graphs. The environment it handed to a step, the per-visit
  token, and the routes that token unlocked are removed, so nothing a tenant writes runs with the hub's reach.

## Left for later

Delivery logic a tenant writes for itself, run where the tenant already trusts its own code — a command step on its own
runners, or a service of its own that the flow's landed event calls — waits for a tenant who needs one.
