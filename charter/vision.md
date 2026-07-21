# Vision

## The fleet we want

Picture a development team whose junior members never sleep. One or more coding agents — Claude Code today, OpenCode and Codex later — run continuously on local machines, pulling work from a shared queue, executing each unit in its own isolated environment, and delivering the results: merged to the main branch in the baseline, or parked at a human-review gate where one is configured. A human hears from the fleet only when an agent is genuinely blocked.

The fleet keeps working overnight and unattended. It recovers cleanly from crashes and reboots. When an agent gets stuck, a human steps into that agent's full context in one motion — and steps back out just as easily. And it scales the way a career does: from a single developer running a handful of agents on one machine to a small team coordinating across several.

## Why winter is the key binding

Blizzard itself is workspace-agnostic — the workspace is a seam, like the forge and the harness ([mission](./mission.md)). But an orchestrator is only as capable as the chunks of work its agents can safely hold, and that is why winter matters so much as the reference binding.

Winter is what makes a single agent powerful. A winter feature environment composes one git worktree per project repository, all on a shared branch, with its own ports and its own running services. That is the machinery behind the flexible work chunk: one agent takes a chunk of one or more tasks, fans out subagents across one or more feature environments, and touches one or more repositories — with isolation guaranteed at every level. Two agents on different chunks never share a working tree, never collide on ports, never trip over each other's services. There is nothing to lock and nothing to serialize.

The rest of the ecosystem cannot express this. Off-the-shelf orchestrators assume one repository, one worktree, one branch, one agent — the rigid shape the mission forbids. Winter is what makes the many-to-many chunk executable at all: not a foundation blizzard is welded to, but the workspace that gives blizzard's agents the most room to work.

## Deterministic shell, intelligent core

One design principle runs through everything blizzard is:

> All loop, arbitration, and recovery logic is boring deterministic code. LLMs supply judgment only at well-defined points.

The control plane — the queue, the lease protocol, the reconciliation loop, the fencing, the crash recovery — is ordinary deterministic software: testable, `kill -9` safe, and never dependent on a model behaving well. Models are invited in only where judgment is genuinely the job — writing the code, reviewing and testing it, optionally planning how open work should be shaped into batches, and rendering the verdict a finished worker hands back.

The reason is easy to hold: an LLM can be wrong in judgment, and the system is built to survive that. It is never handed a lever that lets it be wrong in arithmetic.
