# The delivered Claude Code adapter

> **Shipped.** This plan is frozen: it records the adapter foundation delivered with `milestone:mvp`, and its entry in
> [delivered.md](../../delivered.md) is the record of what landed.

Blizzard drives coding agents it does not own, through CLIs that change monthly. The adapter is the shim that makes this
survivable: one small interface per harness, kept deliberately **dumb** — an adapter translates, it never decides.

## What to build

- **Four operations per harness.** Spawn a headless worker into a chunk's environments; resume a session headlessly with
  a message — the single operation behind the judgement prompt, answer delivery, and future CI feedback; compose the
  literal interactive-resume command a human can paste; and parse the judgement reply into a structured verdict.
- **Hooks, delivered at spawn.** The worker's hook set travels with the adapter, never materialized into project repos:
  a heartbeat on every tool call, and a deny-with-guidance on the harness's native ask-the-user tool. Identity is
  runner-minted and echoed back, never self-reported.
- **Enforcement at the boundary, not in hooks.** Gating strength varies by harness, so nothing correctness-critical
  depends on hooks: guarded resources sit behind a harness-agnostic wrapper, and hooks stay polish.
- **Human takeover.** An escalated chunk carries the literal resume command that lands a human in the stuck agent's full
  session, with `blizzard runner takeover` as the ergonomic path that composes it correctly. Hand-back is an explicit
  requeue; the interactive session runs outside supervision.
- **Adapters kept honest.** Pinned harness versions, a per-harness conformance suite, and `blizzard runner selftest` put
  a trivial task through the harness before unattended operation.
