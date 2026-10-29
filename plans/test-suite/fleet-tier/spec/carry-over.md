# Carry-over

The epic adds reads to the hub and the runner, and one behaviour to the hub: logging out ends a session's streams. Every
addition is additive to each supported release. Nothing an installation holds moves, and `blizzard-mock` is unchanged.

## The order of landings

1. **The support modules** ([retirement.md](./retirement.md#the-support-modules-first)): the helpers other tiers borrow
   move out of `tests/e2e/`, and the helpers e2e files borrow from one another move into `tests/e2e/support.py`, before
   any file is ported.
2. **The speed guard's sizing** ([below](#the-speed-guard)), before the first port.
3. **The UI method, task, and required job** ([harness.md](./harness.md#the-ui-project)), before the first browser
   scenario leaves e2e.
4. **The harness's gains** ([harness.md](./harness.md)): the `actions` profile and its owned environment, trace
   companions, the sink's span decoding and the ported shape checks, the collector on the `playwright` task and in CI,
   the bare-origin reads, and the DuckDB and file fixtures. Each lands with the first port that needs it, after the
   runner tier's `service-fleet` projects.
5. **The surfaces** ([surfaces.md](./surfaces.md)), each in the change of the first port that needs it, or ahead of it.
   Logging out ending a session's streams lands on its own, with component tests, before the login port.
6. **The moves down and the drops** ([roster.md](./roster.md#what-moves-down)), each in a change of its own, any time.
7. **The ports**, one change per file: `service/fleet/` first, then `ui/`, the split files' halves each in the change of
   the folder it lands in.
8. **The retirement** ([retirement.md](./retirement.md#what-names-the-tier)), when the last file has moved.

## Citations and the rulebook

Each port moves its citations under `bzh:port-moves-citations`, and moves its scenarios from blizzard-context's e2e
roster to the page of the method that took them, in the same change, so the registry's roster check stays green
throughout ([retirement.md](./retirement.md#what-names-the-tier)).

## The speed guard

The e2e tier runs today in a job of its own, so its thirty-five minutes hold up no other check. Its scenarios join
`service-fleet`, which runs in the Playwright job beside the rest of the service tier, so the initiative's guard holds:
the tier never becomes the gate's slowest job. The cost is sized before the first port, not discovered after. A profile
project runs one worker on one hub, so ideation, the event log, hub actions, and the gardening dialog run in sequence on
`actions`; each of the nine trace companions runs a whole scenario again on a fresh hub; and the Playwright job's
timeout is thirty minutes. The sizing estimates each project's time from the e2e run's per-file costs and the runner
tier's measured host cost, about 6.6 seconds to initialize and 13 for a whole test, and decides the job's shape before
the first port: how many shards `service-fleet` takes, and whether `actions` and the companions run in jobs of their
own. Each port then records its project's CI time against that estimate, and a port that would cross the line is
preceded by the change that reshapes the jobs.
