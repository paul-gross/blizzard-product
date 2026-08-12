# Plan — `epic:board`, chunk-detail slice

A chunk's artifact store is where the fleet's real output accumulates — plans, diffstats, review findings, pinned
commits — and the board currently pours all of it, verbatim, into a small dock panel. A findings transcript hundreds of
pages long renders inline, burying every artifact after it and the detail sections below. The material is right; the
surface is wrong: long-form artifact text needs a page, not a panel.

## What to build

- **The dock names, the page shows.** The board dock's artifact section becomes a list of links — key, kind, recency —
  with no inline asset content. A pinned commit may keep its one-line `repo @ commit` rendering; prose never renders in
  the dock.
- **A chunk detail page.** A routed page per chunk with two tabs: **General** — identity, status, pointers, node
  history, the dock's existing sections given room — and **Artifacts** — a navigation list of the chunk's artifact store
  beside a full-height scrollable viewer.
- **Links land pre-selected.** A dock artifact link opens the page on the Artifacts tab with that artifact selected, so
  the operator's click goes straight from "what is this?" to reading it.
- **One renderer.** The per-artifact rendering stays with its single owner (the shared artifact body component the
  mobile shell already composes); the page composes it rather than re-typing the kind branch.

## Artifacts

| Mockup                                                       | What it explores                                                                                                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| [chunk-detail-page.html](./artifacts/chunk-detail-page.html) | The page: General/Artifacts tabs, the artifact nav beside the full-text viewer, and the dock's compact link row that replaces inline text. |
