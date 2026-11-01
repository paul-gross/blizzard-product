---
epic: adapters/opencode-2
refinement: scaffolded
status: horizon
---

# Plan — `epic:adapters/opencode-2`

An operator who has moved to OpenCode 2 finds the fleet will not follow them there. The OpenCode binding admits only the
1.x line it was proven against, so a runner holding OpenCode 2 reports the harness as version-incompatible and never
offers it to a session. That refusal is the binding doing its job: OpenCode 2 is a new build rather than a point
release, and nothing the 1.x proof observed — how a turn is spawned and resumed, which process owns it, which events
carry final text and usage, how plugins and permissions load, how a transcript reads while the worker is live — can be
assumed to have survived the rewrite.

So the epic starts where the [OpenCode epic](./opencode/index.md) started, with the real CLI rather than with the code.
The compatibility proof runs again against OpenCode 2 with every row open: a behavior that still holds re-enters as a
fixture, a behavior that moved becomes a new contract the binding parses, and a behavior with no analogue becomes a
declared degradation, visible as absent. Only once that result exists do the execution, transcript, and hardening steps
follow, each scoped to what actually changed rather than rebuilt by rote. The binding then admits OpenCode 2 beside 1.x,
and a runner advertises whichever it can actually operate.

OpenCode 1 and OpenCode 2 are supported at the same time, and that is the destination, not a migration window. A runner
with either one is a first-class worker, and the fleet's runner host keeps running 1.x throughout. This epic adds 2.x
and retires nothing: 1.x stays admitted, and no removal is planned beside it. Whether OpenCode 2 lives inside the
existing `opencode` harness id with the binding choosing by observed version, or earns an id of its own, is the first
decision the proof should settle — and with it, whether a session minted under 1.x may be resumed by 2.x at all, or must
stay with the version that began it.

This epic is built by hand on an operator workstation that has OpenCode 2, not by the fleet. No fleet runner host
carries OpenCode 2, and the proof, and every step after it, needs a live OpenCode 2 to be honest. Its work stays outside
the fleet's intake until a runner host does.
