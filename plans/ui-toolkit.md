# Plan — `epic:ui-toolkit`

Every small surface the board grows — a form, a picker, a confirmation sheet — is invented, styled, and maintained by
hand, and the count climbs with every epic that touches it. The cost of writing each one is the visible half. The
expensive half is drift: two dialogs written six months apart agree on nothing, and the operator learns each of them
separately. Assembling the next one from a toolkit is faster to write, and quieter to use. In the spirit of
`epic:config`, this is a survey first and an adoption second.

## What to build

- **The survey.** A deliberate look at the component libraries and frontend accelerants the web surface does not yet
  use, judged against the board's real needs — the forms, tables, sheets, and pickers it already has and the ones its
  roadmap will ask for.
- **One choice, made on evidence.** The candidate that wins does so on a demonstration rather than a feature matrix, and
  the reasoning is written down where the next person to ask can find it.
- **Migration where the payoff is real.** Hand-built pieces move to the toolkit when the move buys something; the rest
  stay. Wholesale replacement is not the goal and would not pay.
- **A path for the next surface.** After adoption, the way to add a form is to assemble one — demonstrated by building
  the next real surface the board needs from parts.

## Open questions

- Whether a choice worth making constrains the existing frontend stack, and what an honest answer costs if it does.
- What counts as evidence: the most convincing experiment is building one real board surface twice, which is also the
  most expensive.
- How the epics already queued against the board — the story map, `epic:intake`, and `epic:self-sourced-work`'s
  authoring surface — sequence around this one, since each is cheaper after it and blocked waiting for it.
