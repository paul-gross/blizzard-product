# Storybook specifications

The technical contracts behind `epic:storybook`. Product intent begins at the [epic plan](../index.md): which parts the
workshop shows and why the kit comes first, that stories are for people and assert nothing, the toolchain's accepted
costs, and that the workshop lands before `epic:ui-toolkit` judges its candidates in it. This spec states how each of
those is built and redefines none of it.

Paths are relative to the `blizzard` repository's root unless a file names another repository.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, `architecture/frontend-structure/kit.md` gains
`bzh:frontend-kit-stories` beside `bzh:frontend-kit-floor`. `verification/blizzard.md` gains the command method
`web:story-render` and the manual method `web:manual-workshop`, and its spokes take them up: `commands/web/` describes
the render check and the structural gate's new sweep, `manual-web.md` carries the workshop method, `tier-rules.md`'s
`bzh:visual-change-needs-a-render` admits a workshop render, and `tools.md`'s `tool:service-up` names the workshop among
the services it starts. In the winter workspace that builds blizzard, `.winter/config.toml` gives the workshop a port in
every feature environment's band, and `winter-service-tmux`'s manifest and layout hook give it a service and a pane
([workspace.md](./workspace.md)). In `blizzard`'s own docs, `web/README.md` gains the workshop beside the dev servers.

| Where                            | Read when                                                                                                                       |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| [toolchain.md](./toolchain.md)   | Installing and configuring the workshop — the framework, the install, the build, the styles, the providers, the scripts         |
| [stories.md](./stories.md)       | Writing a story — where it lives, what a part owes, the roster of kit parts and composed pieces, and the lens control's reshape |
| [builders.md](./builders.md)     | Building the wire-shape builders in `fleet/testing`, or moving a spec onto them                                                 |
| [gates.md](./gates.md)           | Holding the workshop — the kit's stories sweep, stories among the existing sweeps, the render check, and the rule behind them   |
| [workspace.md](./workspace.md)   | Serving the workshop in every feature environment, and the manual method that makes it part of frontend work                    |
| [carry-over.md](./carry-over.md) | Landing the epic — the order of its pieces, and what each one leaves the next                                                   |
