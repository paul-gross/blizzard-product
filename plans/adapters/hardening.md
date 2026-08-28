# Operational hardening

A second adapter is first-class only when the operator can trust the fleet to notice when it stops being compatible.
OpenCode therefore joins the same preflight and continuous evidence as Claude Code: an observed version, a conformance
run, a real edit-and-commit selftest, and diagnostics that remove a failed capability from acquisition instead of
letting work discover the failure after claim.

The normalized OpenCode dialect also becomes legible to analytics. Tool names and argument shapes are registered under
the normalizer version that produced them, so reads, skill invocations, and child-agent work remain comparable without
pretending the raw harness records are identical. Every lease, invocation, escalation, and transcript segment records
the actual harness provenance needed to compare quality and cost later.

Subscriptions leave the adapters here. What an operator pays for is a plan with a provider, not a CLI: one OpenAI
subscription feeds both OpenCode and Codex, so asking each harness to report on it would count the same window twice, or
leave it unreported on a night the harness that owned it never ran. The runner declares the subscriptions it holds, each
under a slug it is known by and a name the operator reads, and a runner loop samples each one on its own cadence by
whatever means that provider offers. Harnesses spend a subscription; they do not report it.

None of that arrives by breaking what already reports. The new per-subscription record grows beside the single window
the runner sends today, which keeps being sent, so a fleet mid-upgrade never has a blind night — and the operator meets
the change as a panel that can finally show two plans at once rather than as a number that stopped arriving.

Hardening is also where unequal capabilities are made ordinary. OpenCode's model variants may not share one universal
effort vocabulary, and its compaction controls may not express Blizzard's numeric window. Those absences are surfaced as
unsupported, tested as such, and kept out of correctness paths.

The board catches up here too. A runner stops being one machine with a capacity and becomes a machine with harnesses,
each shown with the version it observed and whether it is actually available — so an operator can tell a runner that is
idle from one that is idle because its OpenCode binding failed its selftest. Work carries the harness it ran under into
the surfaces where its model and effort are already read.

The slice closes when a runner can advertise Claude Code and OpenCode together, route a compatible graph
deterministically, survive restart and interruption through both, and run unattended without either adapter borrowing
assumptions from the other. The [hardening specification](./spec/hardening.md) owns the verification and degradation
contract.
