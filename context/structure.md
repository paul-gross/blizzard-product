# Repository structure

What lives where in `blizzard-product`, and which folders are allowed to exist. A file that fits none of these rows belongs somewhere else — most likely in a GitHub issue (work that is ready), or nowhere (`winter-canon:/organization.md`, `canon:admission-test`).

## The layout

| Path | What belongs there |
|------|--------------------|
| `roadmap.md` | The epic registry: the horizon, in-flight, and shipped tables. The single home of every `epic:<slug>` id. |
| `epics/<epic-slug>.md` | The requirements docs — one per promoted epic or slice, named for the epic id it elaborates (`epics/security-worker-lockdown.md` for a slice, `epics/ci-feedback.md` for a whole epic). Written at promotion time, cited by the issues that decompose it. The folder materializes with the first promotion. |
| `context/` | These authoring conventions — the structure and the voice. Agent-facing, read on demand. |

## Deliberately absent

These folders are missing on purpose. Their absence is a design decision, not an oversight — do not introduce them.

- **Work-tracking folders of any kind** — blizzard's ready queue is GitHub and its execution record is closed issues plus the roadmap's Shipped table; mirroring that state in directories would create a second source of truth to drift.
- **Design or decision logs** — the historical record is the frozen `blizzard-discovery` corpus. This repo states current intent only; when intent changes, its files change, and git history is the log.

## Admitting something new

A new folder earns its place only when a class of files exists that no current row can honestly hold, and admitting it means updating the table above and the root `index.md` routing in the same change (`canon:index-scrutiny`). One file is not a class — park it at the root until it has siblings.
