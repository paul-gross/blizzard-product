# Writing guide

Every product document in this repo speaks in one voice, and this file defines it. It governs the documents that state intent — registry entries, plans, capability descriptions. Convention files like this one, and CONTRIBUTING, stay in the workspace's plainer instructional register; the canon's mechanical principles (`winter-canon:/principles.md`) apply everywhere.

## The voice

Write as a well-tenured product owner — someone six to ten years into the craft — who speaks elegantly about non-technical concepts, with a vocabulary slightly advanced for their level of experience. Their signature skill is finding the expression of a concept that every level of developer can understand: the phrasing that lets a first-year engineer and a principal architect walk away holding the same idea.

That person has a few habits worth imitating:

- **They start from the person with the problem.** A capability is introduced through who needs it and what their day looks like without it, before any mention of what will be built. If a paragraph could open "the system will…", it should probably open with the operator, the reviewer, or the person on call.
- **They trust complete sentences.** Ideas arrive as prose, in an order a reader can follow, with one idea handed to the next. Lists appear when items are genuinely parallel and enumerable — never as a substitute for thinking through how ideas connect.
- **They reach for the concept before the mechanism.** "The fleet should never be able to spend more overnight than its owner would forgive by morning" comes first; token accounting and kill-switches come after, and only where the detail changes a decision.
- **They choose the occasional exact word.** *Provenance*, *cadence*, *forgiving*, *appetite* — a slightly elevated word is welcome when it is the precise one, and suspect when it is decoration. Elegance here means economy, not ornament.
- **They build bridges instead of lowering the ceiling.** When a concept risks losing part of the audience, they add an analogy or a concrete moment ("the morning after a bad night, what does the operator wish they could see first?") rather than flattening the idea. Simplifying the expression, never the thought.

## What this voice is not

The failure mode to guard against is the default register of the models that write most of these drafts. The tells, so they can be caught in review:

- Bullet cascades where prose should carry the argument, and sentence fragments posing as sentences.
- Sales adjectives — *powerful*, *seamless*, *robust*, *comprehensive* — which describe the author's enthusiasm, not the product.
- Formulaic symmetry: the triad ("fast, safe, and reliable"), the mirrored construction ("not just X, but Y"), the paragraph that restates its own heading as an opener.
- Hedging in layers ("could potentially help enable…") where a product owner would simply say what they believe and own it.
- Jargon without a bridge. A term of art is fine; a term of art doing the work of an explanation is not.

## A calibration example

The same requirement, in the register to avoid and the one to hit:

> **Avoid:** Robust cost controls are essential for fleet operations. The system will implement per-chunk and per-night budget caps, comprehensive token/cost telemetry, and a spend kill-switch, enabling operators to seamlessly manage spend.

> **Hit:** An overnight fleet spends money while its owner sleeps, so the owner needs to decide in advance how much a night is allowed to cost — and trust that the fleet will stop itself at that line rather than explain itself in the morning. Caps per chunk and per night set the line; a kill-switch is the hand on the cord when trust runs out.

The first names features. The second names the person, the fear, and the promise — and a junior developer, an executive, and the engineer implementing the cap all leave with the same picture.
