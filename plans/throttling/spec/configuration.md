# Configuration contract

## Where the levers live

Every throttle is configured under its runner's `[[runner]]` entry in `blizzard-host.toml`, in one new table,
`[runner.throttle]`, beside the breakers' `[runner.cost]`. A lever is a sub-table of it, and a lever whose sub-table is
absent does not exist: it is not asked, it says nothing, and it shows nothing on the runner's status. A runner with no
`[runner.throttle]` at all is held back by its subscription's floor alone ([exhaustion.md](./exhaustion.md)), which has
no key, since nothing about it is the operator's to set.

The file is the only place a lever is set. No route, CLI verb, or panel control writes one, and the settings never leave
the machine: the hub learns which lever is declining, never what a lever is set to ([surfaces.md](./surfaces.md)). Like
every key of `blizzard-host.toml`, the table is a process setting read when the host starts (`bzh:config-key-declared`),
and an edit takes effect at the host's next start.

```toml
[[runner]]
id = "rn_01M3MY1NGQR8WE1ZBRKNREFNE4"
name = "r-claude"
subscription = "anthropic"
projects = [{ project = "blizzard", workspace = "blizzard-claude", agents = 2 }]

# The breakers: each trips, and only a person clears it.
[runner.cost]
runner_ceiling_usd = 60
window_hours = 24
chunk_cap_usd = 15

# The throttles: each declines while it says no, and lifts itself when it says yes again.
[runner.throttle]
gate_type = "hold"             # or "release" — what a chunk the gate parks between steps does
usage_limit_gate_type = "hold" # or "release" — what a lease a usage limit stops mid-step does

[runner.throttle.pace]
window = "7d"                  # a window label the subscription's samples report; its longest when absent
lead_pct = 5                   # points spend may run ahead of the window's elapsed share

[[runner.throttle.pace.taper]]
from_pct = 90                  # once 90% of the window is spent…
lead_pct = 1                   # …spend may run only 1 point ahead

[runner.throttle.utilization]
below_pct = 95                 # work only while every named window is under this
windows = ["5h", "7d"]         # every window the samples report when absent

[runner.throttle.budget]
usd = 40                       # set below the ceiling it stands beside
window_hours = 24

[runner.throttle.schedule]
days = ["mon", "tue", "wed", "thu", "fri"]
hours = ["19:00-08:00"]
```

## The keys

| Key                                       | Type            | Default               | Meaning                                                                                                                                                                                                              |
| ----------------------------------------- | --------------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `runner[].throttle`                       | table           | absent                | The runner's throttles. Absent, only the subscription floor holds it back                                                                                                                                            |
| `runner[].throttle.gate_type`             | string          | `"hold"`              | `"hold"` keeps a chunk the gate parks between steps; `"release"` puts it back in the queue for whichever runner claims it next ([release.md](./release.md))                                                          |
| `runner[].throttle.usage_limit_gate_type` | string          | `"hold"`              | `"hold"` keeps a lease a usage limit parks mid-step until the floor opens; `"release"` puts its chunk back in the queue, abandoning the half-done step ([release.md](./release.md) §Releasing a usage-limited lease) |
| `runner[].throttle.pace`                  | table           | absent                | The pacing lever ([levers.md](./levers.md) §Pacing)                                                                                                                                                                  |
| `runner[].throttle.pace.window`           | string          | the longest reported  | The label of the window paced against, as the subscription's samples name it (`5h`, `7d`)                                                                                                                            |
| `runner[].throttle.pace.lead_pct`         | number          | `5`                   | How many percentage points the window's spent share may run ahead of its elapsed share, `0` to `100`                                                                                                                 |
| `runner[].throttle.pace.taper`            | array of tables | absent                | Steps that narrow the lead as the window fills ([levers.md](./levers.md) §Pacing)                                                                                                                                    |
| `runner[].throttle.pace.taper[].from_pct` | number          | required in a step    | The spent share, `0` to `100`, from which the step's lead applies                                                                                                                                                    |
| `runner[].throttle.pace.taper[].lead_pct` | number          | required in a step    | The lead, `0` to `100`, in force from `from_pct` up to the next step                                                                                                                                                 |
| `runner[].throttle.utilization`           | table           | absent                | The utilization line ([levers.md](./levers.md) §The utilization line)                                                                                                                                                |
| `runner[].throttle.utilization.below_pct` | number          | required in the table | The line, `0` exclusive to `100` inclusive: the lever says yes only while every named window is under it                                                                                                             |
| `runner[].throttle.utilization.windows`   | list of strings | every reported window | The window labels the line applies to                                                                                                                                                                                |
| `runner[].throttle.budget`                | table           | absent                | The dollar budget ([levers.md](./levers.md) §The dollar budget)                                                                                                                                                      |
| `runner[].throttle.budget.usd`            | number          | required in the table | The budget, a positive figure counted as [dollars.md](./dollars.md) counts every line                                                                                                                                |
| `runner[].throttle.budget.window_hours`   | number          | `24`                  | The rolling window the budget is summed over, positive                                                                                                                                                               |
| `runner[].throttle.schedule`              | table           | absent                | When the runner is willing to start work ([levers.md](./levers.md) §The schedule)                                                                                                                                    |
| `runner[].throttle.schedule.days`         | list of strings | every day             | Lower-case three-letter day names, `mon` to `sun`                                                                                                                                                                    |
| `runner[].throttle.schedule.hours`        | list of strings | the whole day         | `HH:MM-HH:MM` spans on a 24-hour clock; a span whose end is not after its start runs past midnight                                                                                                                   |

Every key is declared in `runner/config_keys.py` beside the `[runner.cost]` keys it sits under (`:190`–`:193`), so
`contracts/config/host.json` gains the eighteen entries when it is regenerated (`mise run export-contracts`) and
`blizzard:config-contract` holds the document to the registry. `[runner.cost]` keeps its three keys and their meaning as
breakers; only how its dollars are counted changes ([dollars.md](./dollars.md)).

## Parsing

The levers are a domain value of the `throttle` package, `ThrottleSettings` in `runner/throttle/settings.py`: one
optional record per lever — `PaceSettings`, carrying its base lead and its taper as an ordered tuple of `TaperStep`s,
`UtilizationSettings`, `BudgetSettings`, `ScheduleSettings` — and the `gate_type` and `usage_limit_gate_type` choices,
each a `GateType` enum, `HOLD` or `RELEASE`. The config edge builds it: a `Throttle` section parser in
`runner/config.py`, beside `Spend` (`:188`), reads the table through the strict view, and `_own_tables` in
`runner/host_config.py` (`:1003`) places it on the runner's `RunnerConfig` as `throttle`, exactly as it places `Spend`'s
three values today (`:1011`, `:1026`–`:1028`). No concept package imports `RunnerConfig` (`bzh:domain-package-layers`):
the gate takes `ThrottleSettings` by injection ([gate.md](./gate.md)).

A schedule's spans are parsed to minutes of the day once, at load, so nothing the gate does on a tick parses text. The
host-file renderer that graduation and ingest use (`runner/host_config.py:1159`), which writes `[runner.cost]` back out
for a runner it moves, writes `[runner.throttle]` and its sub-tables the same way, so a runner ingested into another
host keeps its throttles.

## Validation

Every throttle error is a runner error in `HostConfig.load`'s sort
([`delivered/runner-host`'s configuration contract](../../../delivered/runner-host/spec/configuration.md) §Validation):
it halts that runner alone, names the key, and `blizzard runner host --check` reports it with the rest. They are:

- a `gate_type` or `usage_limit_gate_type` other than `"hold"` or `"release"`;
- a `pace` or `utilization` table on a runner that names no `subscription`, or whose subscription's `provider` binds no
  sampler (`select_sampler`, `runner/subscriptions/internal/subscription_sampler_factory.py:24`) — a lever that judges
  by samples, on a runner whose plan is never sampled, could only ever say no;
- a taper step missing `from_pct` or `lead_pct`, or either outside `0`–`100`, or steps whose `from_pct` values are not
  strictly ascending in the order written;
- a `lead_pct` outside `0`–`100`, a `below_pct` outside `0` exclusive to `100` inclusive, or an empty `windows` list;
- a `budget` with no `usd`, a non-positive `usd` or `window_hours`, or a `usd` at or above the runner's
  `runner_ceiling_usd` when both are summed over the same number of hours, since that ceiling would trip before the
  budget ever throttled;
- a schedule naming an unknown day, a span that does not parse, a span of zero length, or a table with neither key.

A window label is not validated at load: the labels are the provider's, carried on each sample, and `openai` labels its
windows from the lengths its own response reports (`docs/deployment/spend.md` §External subscription usage). A `window`
or `windows` entry no sample names makes its lever say no, naming the label ([levers.md](./levers.md)), and the runner's
status shows that lever's answer beside its setting ([surfaces.md](./surfaces.md)).
