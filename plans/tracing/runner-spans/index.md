# Plan — `epic:tracing`, runner-spans slice

The hub sees a node step from a distance. It knows when the runner said the attempt began and when the runner reported
how it ended, and [fleet-spans](../fleet-spans/index.md) tells the step from exactly that. Everything between those two
reports happened on the runner, and the hub never saw it. Within one step, the agent may have been launched, run for
forty minutes, been resumed after a question was answered, been nudged when it went quiet, and been asked to judge its
own work. It may have waited out a provider that was turning requests away, been parked while an operator paused the
chunk, or been taken over by a person at a terminal. Every one of those moments is in the runner's record already. A
person looking at the step sees one undivided bar, and the runner is the only place that knows how the bar divides.

[`persona:harness-engineer`](../../../charter/personas/harness-engineer.md) feels this most. A step that took twice as
long as it should have might have been one slow session or three short ones and a long resume, a judgement that ran
away, or twenty minutes lost to a provider's rate limit. Those causes call for different fixes, and today telling them
apart means reading the runner's store by hand.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

**The step's inside, from the runner that ran it.** Each runner tells the steps it ran from its own record. It adds them
to the same traces the hub tells, with no message between the two daemons: the step's identity is derived the same way
on both sides, so the runner's spans land inside the hub's step without either knowing when the other reports. Inside a
step, the runner shows:

- the worker it held for that attempt, with the true times as the runner measured them
- each harness invocation in order: the first launch, every resume, every nudge, and the judgement
- the time the worker sat parked on a question, or parked while the chunk was paused
- each wait out a provider's overload
- any stretch a person spent driving the session by hand
- the checks it ran and whether each passed

**Invocations that read as model work.** Each invocation is described in OpenTelemetry's own vocabulary for an agent
invocation: the model it ran, the tokens it read and wrote, what it cost. A backend's built-in views of model work then
show blizzard's fleet without anyone teaching them blizzard's names. Claude Code and OpenCode report through the same
shape, which is the comparison the [harness engineer](../../../charter/personas/harness-engineer.md) has never had: the
same step, under two harnesses, side by side.

**The runner's account, kept to the runner's facts.** This slice tells what the runner itself recorded, and it stops
there. What the agent did inside an invocation — the files it read, the tools it called, the subagents it started —
stays with whatever the harness reports about itself, as the [epic plan](../index.md) promises.

**Told the same way as the hub's.** The runner tells a step once it has closed, reading its own record forward from
where it last stopped, beside its loop rather than in it. It shares the hub's discipline in full: off until an endpoint
is named, history only on request, never anything that makes a worker wait.

## What it is not

It is not a live view of a running worker. A step's inside appears when the step ends. It sends no content: no question
text, no check output, no transcript, no file path. It does not move the hub's account of when a step began or ended.
The runner's measured times sit inside the step as its own spans, and the difference between the two clocks is itself
something an operator can now see.

## How it is proven

Against `blizzard-mock`, chunks that ask a question, run a review that bounces, escalate, and run under both harnesses
produce runner spans inside the steps the hub tells for the same run, matched by id alone. Each invocation reads as an
agent invocation in a backend's model view. The [specification](./spec/index.md) owns the detail.
