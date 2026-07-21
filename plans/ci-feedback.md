# Plan — `epic:ci-feedback`

When CI goes red or a reviewer asks for changes after a delivery is open, the news has to reach the agent that did the work — and today the messenger is a human. The failure escalates with a resume command, and someone pastes it, relays the log, and shepherds the fix. That human relay is the version that ships; it is correct, and it does not scale past the operator's patience. This epic automates the relay: the result finds its own way back into the owning session.

The delivery machinery this feeds into — the merge queue, per-repo landed facts, gated pull requests — already exists.

## What to build

- **The result routed home.** A red CI run or a change-request lands back in the session that owns the delivery: the dormant session is resumed with the failure in hand — a log excerpt, the review comments — the same way an answered question resumes its asker.
- **The mapping as one durable fact.** Which session owns which delivery is a single stored fact linking a chunk's delivery (its PR, where a gate opened one) to the session that must receive its feedback — one row, not an inference reconstructed from history.
- **A hard cap on the loop.** Feedback rounds are capped at two. An agent that fixes one thing and breaks another can oscillate indefinitely; the cap turns oscillation into an ordinary escalation, and the human relay — which never stops working — takes it from there.

## Open questions

- What the CI seam watches in the reference stack: polling the forge's check runs, a webhook the hub exposes, or both — and how a self-hosted hub without an inbound surface hears the news.
- How much of a failing run the session is fed: the excerpt heuristic that is informative without drowning the context.
