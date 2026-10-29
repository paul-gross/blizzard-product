# Carry-over

The epic adds reads to the hub, the runner, and the mock forge, and one behaviour to the hub: a revoked session closes
its streams. Every addition is additive to each supported release. Nothing an installation holds moves.

## The order of landings

1. **The support modules** ([retirement.md](./retirement.md#the-support-modules-first)), before any e2e file is deleted.
2. **The harness's gains** ([harness.md](./harness.md)): the `actions` and `egress` profiles, trace companions, the
   sink's span decoding, the collector fixture, and the runner host's environment. Each lands with the first port that
   needs it, after the runner tier's `service-fleet` projects.
3. **The UI project** ([harness.md](./harness.md#the-ui-project)), proven by the first UI port.
4. **The surfaces** ([surfaces.md](./surfaces.md)), each in the change of the first port that needs it, or ahead of it.
   A revoked session closing its streams lands on its own, with component tests, before the login port.
5. **The moves down and the drops** ([roster.md](./roster.md#what-moves-down)), each in a change of its own, any time.
6. **The ports**, one change per file: `service/fleet/` first, then `ui/`, the split files' halves each in the change of
   the folder it lands in.
7. **The retirement** ([retirement.md](./retirement.md#what-names-the-tier)), when the last file has moved.

## Citations and the rulebook

Each port moves its citations under `bzh:port-moves-citations`. The e2e scenario pages in blizzard-context name the test
that took each scenario as it lands, so the pages are empty, and removable, by the retirement.

## The speed guard

The e2e tier is not in the gate, but its scenarios join `service-fleet`, which is, so the initiative's guard holds: the
tier never becomes the gate's slowest job. Each port records the fleet project's CI time. The runner tier's spike puts a
runner host at about 6.6 seconds to initialize and a test at about 13 seconds whole; the heaviest files, the pull
request and CI cases, ideation, and the garden routine, are measured as they land. If a port would cross the line, the
next change, before any further port, shards `service-fleet` across more CI jobs, or reuses one minted fixture world per
worker, whichever the measurement favours.
