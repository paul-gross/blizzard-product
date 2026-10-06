# Plan — `epic:projects`, runner slice

The hub slice teaches the hub which project every chunk belongs to, stores the projects every runner serves, and offers
a runner only those projects' chunks. All that is left for the runner is to say which projects it serves. This slice is
that and no more: a runner keeps its one workspace, exactly as it works today, and an operator who wants projects worked
apart runs more runners.

A machine serving many projects from one runner host, with a workspace for each, is the epic's later `host` slice, built
on `epic:runner-host`. This slice needs neither.

## What to build

- **A runner names the projects it serves.** The runner's configuration gains a list of projects, by id or by slug —
  usually one, sometimes several that share its workspace, as `blizzard` and `blizzard-context` share one workspace that
  holds both their repositories. The runner sends the list at every registration, and the hub slice narrows its queue
  and rechecks its claims against it, so declaring is all the runner has to do to be offered the right work.
- **A new runner starts out serving `default`.** `runner init` writes `projects = ["default"]` into the configuration it
  scaffolds unless told otherwise, and every tenant holds a project of that slug from its creation. A freshly set-up
  runner — a feature environment's, the mock fleet's, a test's — is offered work on its first registration with no edit,
  and an operator with a second project changes one line.
- **Everything else about the runner stays as it is.** It keeps its one workspace, its own list of repositories, its
  environments, and its prompt. Which repositories a chunk's work may land in is decided where it always is, at landing,
  where the hub refuses to deliver into a repository the chunk's project does not link.
- **A runner shows what it serves.** Its status names the projects it declares, and marks one that no longer resolves —
  a retired project, or a slug that moved — as the board already marks an unresolved declaration.
- **The existing runner, carried over.** A runner whose configuration names no projects sends none, and the hub keeps
  the declaration its carry-over recorded — the `default` project — so no deployed runner needs a configuration change
  until its operator wants a second project.
