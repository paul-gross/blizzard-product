# Plan — `epic:adapters`

> **Shipped** (Claude Code binding). This plan is frozen: it records what the harness adapters were conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed. The remaining harness bindings stay future scope.

Blizzard drives coding agents it does not own, through CLIs that change monthly. The adapters are the shim that makes this survivable: one small interface per harness, kept deliberately **dumb** — an adapter translates, it never decides. All three target harnesses (Claude Code first, OpenCode and Codex designed-for) converged on the same shape — headless run, persisted session, resume — so one interface covers them all, and the less an adapter does, the less there is to break.

## What to build

- **Four operations per harness.** Spawn a headless worker into a chunk's environments; resume a session headlessly with a message — the single operation behind the judgement prompt, answer delivery, and future CI feedback; compose the literal interactive-resume command a human can paste; and parse the judgement reply into a structured verdict. That is the whole contract.
- **Hooks, delivered at spawn.** The worker's hook set travels with the adapter, never materialized into project repos: a heartbeat on every tool call, and a deny-with-guidance on the harness's native ask-the-user tool — a headless worker has no attached human, so a fumbled question is redirected to `blizzard runner ask` in the moment it happens. Identity is runner-minted and echoed back, never self-reported.
- **Enforcement at the boundary, not in hooks.** Gating strength varies by harness, so nothing correctness-critical depends on hooks: guarded resources sit behind a harness-agnostic wrapper, and hooks stay polish.
- **Human takeover.** An escalated chunk carries the literal resume command that lands a human in the stuck agent's full session — their worktrees, their context — with `blizzard runner takeover` as the ergonomic path that composes it correctly. Hand-back is an explicit requeue; the interactive session runs outside supervision, and that is by design.
- **Adapters kept honest.** Pinned harness versions, a per-harness conformance suite, and `blizzard runner selftest` — a trivial task through each harness before any unattended period, so a drifted CLI is caught before it costs a night of work.

## What falls out for free

Once the interface is uniform, a heterogeneous fleet costs nothing: route chunks per harness by fit, or retry a twice-failed chunk under a different harness before a human ever hears about it.
