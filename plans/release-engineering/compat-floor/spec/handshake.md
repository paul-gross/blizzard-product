# Version handshake

## What the runner sends

Every runner registration (`POST /api/fleet/runners`, `RunnerRegistrationRequest` in `src/blizzard/wire/runner.py`) and
every host report (`POST /api/fleet/host`, `src/blizzard/hub/api/fleet_host.py`) gains one optional field:

```python
#: The blizzard version this process runs, as `blizzard.__version__` reports it; ``None`` from a runner that predates
#: the handshake.
version: str | None = None
```

The runner and its host fill it from `blizzard.__version__` (`src/blizzard/__init__.py`), the installed distribution's
version. All of a host's runners run in one process, so they send one version; the hub still judges each registration on
its own, since registration is what the hub gates. The field is optional, so a runner from before the handshake parses
and sends the body it always did (`bzh:fleet-wire-additive`).

## The rule

`src/blizzard/wire/version_window.py` holds `admits(hub: Version, runner: Version, window: VersionWindow) -> Admission`,
the one statement of the window in code. Versions parse as PEP 440 through `packaging.version`, and only the release
segment's `MAJOR.MINOR` is compared: a candidate, a patch, a dev build (`0.N.0.devR+sha`), or a local version compares
as its line. A runner is admitted when its line is at or above `floor` and at or below the hub's own line. Anything else
is refused, as `too-old` or `too-new`. A version that does not parse is treated as absent.

An absent version is admitted while the window's `floor` is below `handshake_since`, a constant beside the rule naming
the first line that sends a version, and refused once the floor reaches it, since every runner the window still admits
then sends one ([carry-over.md](./carry-over.md)).

A hub never requires a runner newer than itself, and never admits one newer than itself either: a newer runner may call
a route or send a value its hub does not have. So the window's upper edge is the hub's own version and needs no
declaration.

## The refusal

A refused registration answers `409` with a body that extends no existing refusal, since registration has none:

```python
class RunnerVersionRefusal(BaseModel):
    """A runner whose version the hub's window does not admit, naming both versions and the window."""

    reason: Literal["too-old", "too-new", "unversioned"]
    runner_version: str | None
    hub_version: str
    floor: str
    detail: str
```

`detail` is a whole sentence an operator can act on: *runner 0.8.3 is older than this hub's oldest supported line, 0.9;
upgrade the runner host to 0.9 or later, up to 0.11*, or *runner 0.12.0 is newer than this hub, 0.11.2; upgrade the hub
first*. The hub records the refusal on the runner's registration row — the version reported, the reason, and when — and
the claim gate (`src/blizzard/hub/api/fleet.py`, where an unregistered runner is denied) denies a runner whose latest
registration was refused, so a refused runner never takes new work. Its leases already held run out under ordinary
recovery.

The runner treats the refusal as it treats a refused token under the host: that runner's tick halts, its view on the
host panel shows `detail`, and the host's other runners carry on
([`epic:runner-host`'s isolation rule](../../../../delivered/runner-host/spec/process.md#isolation)). The halted runner
retries registration on its ordinary cadence, so upgrading the hub lifts the refusal without touching the host. A host
report the hub refuses on version leaves the host's runners to their own registrations and shows the refusal on the
host's panel.

## What the hub answers and shows

`RunnerRegistrationResponse` gains `hub_version: str | None` and `window: VersionWindowView | None` (`floor`,
`ceiling`), both `None` from a hub that predates the handshake, so a runner shows on its panel which hub it speaks to
and how far it may trail it. `GET /api/runners` and the board's runners view (the host card `epic:runner-host` built)
show each runner's reported version and one of `in window`, `refused (too-old)`, `refused (too-new)`, or `unversioned`,
so an operator sees which runner to upgrade before any of them fails.

`blizzard-mock`'s `mock_hub` accepts `version` and answers `hub_version` and `window`, and its lever set gains one that
refuses a registration with `RunnerVersionRefusal`, so the runner's halting on it is service-tested against the mock
(`bzh:wire-change-extends-mock`).
