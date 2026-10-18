# The workspace and the manual method

Blizzard is built by agents in its own winter workspace, so the workshop is served there: a service in every feature
environment, at a known address, started by the same `winter service up` that starts the rest of the environment. The
workspace's files are its own repository's, not `blizzard`'s, and the paths here are relative to the workspace root.

## The port

`.winter/config.toml`'s `[env.feature.vars]` gains the band's next free slot, beside the two dev servers:

```toml
# The workshop — Storybook over the fleet kit (band +6). Needs no daemon; it compiles the
# worktree's own web source, so a feature env's workshop shows that env's kit.
BZ_STORYBOOK_PORT = "${WINTER_PORT_BASE+6}"
```

Every feature environment gets its own port from the one template (alpha 4526, beta 4546, and so on). The workshop's
script reads the variable when it is set ([toolchain.md](./toolchain.md) §The commands), so the service passes nothing.

## The service

`.winter/config/winter-service-tmux/config.toml` gains a slot after `web-runner`, in the web window the two dev servers
already share:

```toml
# --- The workshop (Storybook over projects/fleet) ------------------------------
# `npm run storybook` on the band's +6 port. No depends_on: the workshop renders the
# kit from source and reaches no daemon, so it is up even when the stack is not.
[[service]]
name = "storybook"
target = "1.2"
cwd = "blizzard/web"
cmd = "exec npm run storybook"
port = "WINTER_PORT_BASE + 6"

[service.health]
type = "url"
target = "http://localhost:${BZ_STORYBOOK_PORT}/"
```

`layout-hook.sh` splits the web window's second pane to give it a third, titled `storybook`, and the window's comment
names three panes. `winter service up <env> --wait` then starts the workshop with everything else, and
`winter service logs <env> storybook` reads its build output. Provisioning needs no change: the packages arrive with
`npm ci` through the existing lockfile-stamped step, since they are `web/package.json` dev dependencies.

A first start after a fresh install spends time optimising Vite's dependencies. The health probe is the URL's first
answer, which comes after that work, so `--wait` reports the workshop healthy only when it can be opened.

`blizzard-context`'s `verification/blizzard/tools.md` adds the workshop to `tool:service-up`'s account of the stack: the
workshop at band +6 (`$BZ_STORYBOOK_PORT`), compiling the worktree's current kit and depending on no daemon.

## The manual method

`blizzard-context`'s `verification/blizzard.md` gains a row in Manual testing, and `verification/blizzard/manual-web.md`
gains its section, in the spoke's existing shape:

| Method                | Surface                                                                                              |
| --------------------- | ---------------------------------------------------------------------------------------------------- |
| `web:manual-workshop` | A part of the board or the panel judged on its own in the workshop, in each of its states and widths |

### `web:manual-workshop`

**Surface.** A storied part, from the kit or a shared piece, rendered alone in the workshop: what it shows in each state
its stories carry, and how it answers when a reader changes its inputs through the controls. How a page composes the
part is `web:manual-board`'s or `web:manual-panel`'s. Whether every story still renders is `web:story-render`'s.

**Setup.** The spoke's shared preamble. The workshop is at `http://localhost:${BZ_STORYBOOK_PORT}/`. No seed is needed,
since a story carries its own data. Outside a feature environment, `npm run storybook` in `web/` serves the checkout's
workshop at `http://localhost:6006/`.

**Steps.**

1. Name every part the change touches, and every part that composes one of them and has stories of its own.
2. For each, open its stories and work through every state, confirming the change shows as intended and nothing else
   moved.
3. Push on each changed input through the controls, including values the stories do not hold: an empty string, a long
   one, a list of one, a list of many.
4. Switch the viewport to `phone` and `phone-sm` for any part whose layout answers to width, and to `desktop`.
5. Keep the browser console open throughout; a part that logs an error does not pass.
6. For a visual change, take a screenshot of each changed state at each width the claim names, and keep the screenshots,
   the story ids, and the commit with the change under verification.

**Passes when.** Every changed state renders as intended at every width it is claimed at, every control drives the part
without breaking it, and the console stays clean.

**The workshop while building.** The method is also how frontend work is done. An agent build that changes anything a
person sees on the board or the panel opens the workshop before it writes a part. It looks first for a part that already
does what it needs, and reuses it before inventing another. It shapes its change against the part's stories as it
builds, and before it calls the work done, it performs this method over every part it touched. A change that adds or
reshapes a part brings that part's stories in the same change (`bzh:frontend-kit-stories`), so the next build finds
them.

The matrix's note on the general rows, which says that `web:manual-board` and `web:manual-panel` are the general
exercises, gains a sentence: a change to a storied part names `web:manual-workshop` beside the general row for its
surface, since one judges the part and the other judges the page.

`tier-rules.md`'s `bzh:visual-change-needs-a-render` admits a workshop render among its real renders: a screenshot of
the changed part's stories taken by `web:manual-workshop`, which is a real Chromium render of the part. It covers a
change to the part itself. A change to how a page lays a part out still needs a render of that page, since the workshop
shows the part without its page.
