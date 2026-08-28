# OpenCode compatibility proof

A harness engineer should not have to discover a changed session protocol by losing a night of work. OpenCode support
begins with a pinned-version proof against the real CLI: the adapter contract is admitted only after the process,
session, event, permission, and transcript behaviors Blizzard would depend on have been observed together.

The proof exercises a fresh headless turn, a resumed turn, synchronous judgement, interactive takeover, forced
termination, and a child-agent run. It establishes which process owns a live turn, when its exit is authoritative, how
the fresh session id becomes known, which events carry final text and usage, how a plugin distinguishes a root session
from a child, and whether conversation data remains readable while the worker is live and after it exits. It also runs
the intended proving model, ChatGPT 5.6 Luna at `max` effort, so provider-specific variant behavior is evidence rather
than an analogy to Claude Code.

This slice produces repeatable fixtures and an explicit compatibility result for each dependency. A supported result
becomes a contract the implementation can test. A missing analogue becomes a declared degradation. An ambiguous result
stops the dependent feature from entering the execution slice; it does not get converted into a hopeful assumption.

The [compatibility specification](./spec/compatibility.md) owns the probes and their acceptance evidence.
