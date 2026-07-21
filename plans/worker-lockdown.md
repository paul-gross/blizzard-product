# Plan — `epic:security`, worker-lockdown slice

An unattended fleet runs agents with the permission prompts turned off — that is the honest stance, because a worker nobody is watching cannot be a worker somebody must approve. But honesty about the stance is not the same as comfort with it: tonight's workers hold real credentials, reach a real network, and push to a real forge, and the only thing between an agent and a catastrophe is the agent's judgment. While blizzard dogfoods itself against real GitHub, that is a live risk, not a hypothetical one. This slice builds the walls: the operator decides in advance what a worker may touch, and the platform enforces it rather than hoping.

The runner-auth slice of this epic — runners signing in to the hub with hub-minted tokens — is already delivered; this slice is about the workers.

## What to build

- **Permission profiles.** Named stances between "prompt for everything" and "skip everything," configured per runner, so the skip-permissions default becomes a choice among choices rather than the only option.
- **Blast-radius limits.** What a worker can reach when it goes wrong: network egress bounds, and credential scoping so a worker holds the least a chunk needs — never the operator's whole keyring.
- **Force-push protection.** The one git operation that destroys history a fleet cannot be allowed to reach for. A worker never force-pushes, structurally.
- **Per-node permission config.** Trust dialed by station, the same way gates are: a build node may deserve looser hands than a deliver-adjacent one, and the workflow graph is where that difference is expressed.

## Open questions

- The enforcement seam: how much lands in the harness adapter (each harness's own permission machinery differs) versus below it, where no harness can opt out.
- What the profile vocabulary is — a small named set blizzard ships, or free-form rules the operator composes.
