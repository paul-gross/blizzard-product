# Plan — `epic:documentation`

Blizzard is public and installable, and its README is the entire published surface — so every question past the first
one costs a clone. An operator standing a fleet up, a graph author deciding what a node may declare, and an engineer
weighing whether blizzard fits their shop all arrive with different questions, and today each of them answers theirs by
reading the repository. This epic gives them somewhere to be sent instead: a documentation site at
`paul-gross.github.io/blizzard-docs/`, structured the way [winter's](https://paul-gross.github.io/winter-docs/) is.
Winter's site is the working pattern to copy rather than re-derive.

## What to build

- **The site, in its own repository.** `blizzard-docs`, built with Astro and Starlight and deployed to GitHub Pages on
  push, so the documentation releases on its own cadence rather than blizzard's.
- **The getting-started path.** One route from nothing to a fleet that has worked a chunk, written for the reader who
  has not decided yet whether to commit an evening to this.
- **The operator and deployment surfaces.** Standing up a hub, running a runner, and what the operator is responsible
  for once both are alive.
- **A browsable reference for the CLI and configuration**, generated from the source of truth wherever it can be,
  because a reference that is written by hand is a reference that will disagree with the software.
- **The graph author's concepts.** What a node may declare and what the platform does with it — the questions a graph
  author currently answers by reading the schema.
- **The site as one file an agent can read.** `starlight-llms-txt`, so an agent reads the whole site as easily as a
  person reads one page.

## Open questions

- How much of the reference is generated versus written, and what check keeps the written part honest as the software
  moves.
- What stays in the repository README once the site exists, which should be the shortest possible road to the site.
- Whether the documentation cuts with blizzard's releases or floats ahead of them, and how a reader can tell which
  version they are reading about.
