# Levers contract

Each lever is one pure function in `runner/throttle/levers.py`, taking its own settings
([configuration.md](./configuration.md)), the readings [gate.md](./gate.md) §What it reads gathers, and `now`, and
returning a `LeverAnswer`: whether it admits, the reading it judged in words an operator can check against the board,
and `until` when the same readings already say when it will next admit. A lever never reads a store, the clock, or
another lever. The pace's taper is the only conditional behavior any lever has: no lever's answer or setting depends on
another lever's, and the gate only ever combines answers by requiring every one of them.

## Readings from samples

Pacing and the utilization line judge the newest sample of the runner's one subscription — the windows
`ExternalSubscriptionUsageWindow` carries (`runner/subscriptions/subscription_sampler.py:49`), whose `utilization_pct`
runs `0` to `100` and whose `resets_at` and `window_seconds` place it in time. They are the figures the board's
subscription rows already draw, so a lever and the board never disagree about what was spent.

A sample is fresh while it is younger than three of its subscription's sampling intervals (`sample_interval_seconds`,
300 seconds when absent). A window whose `resets_at` has already passed is not read at all: the window has turned over
since the sample, and what it held no longer describes it. A lever whose named window is missing, unread, or carried
only by a stale sample says no, naming why — `no fresh sample of anthropic since 09:12`, `window 7d not reported` — and
says yes again at the next fresh sample that carries it. It never falls back to an older sample, to another
subscription's, or to a guessed zero.

## Pacing

The pace lever compares how much of one window is spent with how much of it has elapsed. For the named window, or the
longest window the sample reports when none is named:

- the **spent share** is `utilization_pct`;
- the **elapsed share** is `100 × (1 − (resets_at − now) / window_seconds)`, held within `0` to `100`;
- the **effective lead** is the `lead_pct` of the last taper step whose `from_pct` is at or below the spent share, or
  the pace's own `lead_pct` when the spent share is below every step, or when no taper is configured;
- the lever admits while the spent share is at most the elapsed share plus the effective lead.

Five days into a seven-day window the elapsed share is 71; a window 86 percent spent is 15 points ahead and, under the
default lead of 5, declines. The taper lets the lead narrow as the window fills: with `lead_pct = 5` and a step
`from_pct = 90, lead_pct = 1`, the runner may run five points ahead of the week until nine-tenths of the allowance is
gone, and only one point ahead after that, so the last of it is spent no faster than the week passes.

When it declines, `until` is the instant the elapsed share catches up with the spent share less the effective lead —
`resets_at − window_seconds + window_seconds × (spent − lead) / 100` — the moment it would admit were nothing more
spent, which the next fresh sample then confirms or moves. The formula needs nothing new for the taper: while the lever
declines no new step starts, so the spent share holds where it was judged, and is the one the next admission would be
judged at, and the effective lead at that share is the lead the formula takes. The answer reads
`7d: 86% spent, 71% elapsed (lead 5)`, and names the taper step when one is in force:
`7d: 92% spent, 88% elapsed (lead 1 above 90%)`.

## The utilization line

The utilization lever admits while every named window, or every window the sample reports when none is named, stands
under `below_pct`. A window at or over the line declines, and `until` is the soonest `resets_at` among the windows over
it, since a reset is what brings a window back under the line. The answer names the window that refused:
`5h: 96% (line 95%)`.

## The dollar budget

The budget lever admits while the runner's spend over the trailing `window_hours` is under `usd`. The spend is the one
figure every dollar line reads — billed plus estimated, from `usage_since` ([dollars.md](./dollars.md)) — so a runner
billed nothing on a subscription paces in dollars by its estimate exactly as a runner on API keys does by its bill. The
window rolls as the ceiling's does, so a runner that has spent its allowance waits for its oldest spend to age out of
the window rather than for midnight, then carries on. No `until` is given: when enough spend ages out depends on spend
the lever does not read. The answer reads `$41.20 of $40.00 over the trailing 24h`, with the PARTIAL `+` when the figure
is a lower bound.

The budget is a throttle and the ceiling beside it a breaker, and the two read one figure over their own windows. A
runner that keeps spending through its budget — a long step admitted just under the line, or a continuation the budget
does not ask — reaches the ceiling set above it, which trips and stays tripped until a person clears it
(`runner/loop/steps.py:134`). Validation keeps a budget below a ceiling summed over the same hours
([configuration.md](./configuration.md) §Validation).

## The schedule

The schedule lever admits while `now`, read on the host machine's own local clock, falls inside one of its spans. A span
belongs to the day it starts on: `19:00-08:00` on a Friday runs into Saturday morning even when Saturday is not listed,
and does not start on Saturday evening. `days` alone admits those whole days; `hours` alone admits those spans every
day. Spans are wall-clock times, so a span runs by the clock on the wall across a daylight-saving change: a start that
does not exist that night begins at the first minute that does, and a time that occurs twice is taken at its first
occurrence. When it declines, `until` is the start of the next span, and the answer reads
`outside mon–fri 19:00–08:00, next 19:00`.

The schedule decides only whether work starts. A step admitted at 07:58 runs to its end, since a continuation is never
asked about the schedule ([gate.md](./gate.md) §Where it lives).
