# Personas

Every requirement in this repo should be traceable to a person it serves. Each persona carries a stable identifier of the form `persona:<slug>`; other documents cite personas by that id, and a piece of the product that serves no persona here is gold-plating and should be challenged. A persona's card is the id's single home — renaming or removing one is a breaking change for everything that cites it.

[`persona:application-architect`](./personas/application-architect.md) is the primary persona: the engineer blizzard is built by and for. The others are served deliberately later, as the roadmap's layers arrive.

| Persona | In one line | Card |
|---------|-------------|------|
| `persona:application-architect` | The agentic-native engineer who queues work, walks away, and expects to return to landed work | [card](./personas/application-architect.md) |
| `persona:senior-se` | A strong engineer who does not yet trust agents, and needs the fleet to earn that trust gate by gate | [card](./personas/senior-se.md) |
| `persona:product-manager` | Owns what gets built and in what order, and never touches anything below the board | [card](./personas/product-manager.md) |
| `persona:product-owner` | Owns whether what was built is right, and answers the fleet's product questions from their phone | [card](./personas/product-owner.md) |
| `persona:harness-engineer` | The fleet's mechanic, tuning parameters and reading outcomes across many runs | [card](./personas/harness-engineer.md) |
| `persona:fleet-agent` | The worker session itself — the one persona that is not a person, and is trusted as little as possible | [card](./personas/fleet-agent.md) |
