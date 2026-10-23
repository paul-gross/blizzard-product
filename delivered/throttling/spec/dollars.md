# Dollars contract

## One figure

Every dollar line a runner draws — the budget it throttles at, the ceiling and the chunk cap it trips at — reads one
figure: what was billed plus what was estimated, the amount the board already shows for the same work
(`docs/deployment/spend.md` §Estimated cost). The two layers then never disagree about what was spent, only about what
happens at the line. An OpenCode runner on a ChatGPT subscription, billed a literal zero and estimated from OpenCode's
own rates, throttles at its budget and trips its ceiling and its chunk cap on that estimate, as a runner on API keys
does on its bill. What a figure counts is unchanged — billed cost still derives from the harness's own reported figure
by subtraction, and an estimate still from the cached rates of the step it priced — and neither is ever folded into the
other's column: the runner and the hub keep `cost_usd` and `estimated_cost_usd` apart, and every line adds them only to
compare.

## PARTIAL, once more

Once every line reads the board's figure, every line also reads the board's PARTIAL: a figure is a lower bound when some
invocation it sums carried **neither** a billed cost nor an estimate — a worker that died before its envelope, or a
reported figure that ran backwards against its session, with no estimate beside it. An invocation carrying only an
estimate is not a lower bound. This replaces the caps' own reading of PARTIAL as "no billed cost recorded", which made
sense only while they read billed cost alone, and leaves one meaning of the `+` on every surface: the board,
`blizzard hub status`, a pause reason, an escalation's detail, and the runner's own status all mark the same rows.

The hub's `UsageTotal` stays the one owner of that contract (`hub/domain/chunk/model.py:1587`): `cost_partial` is the
reading every line takes. `billed_partial` stays on the wire for the egress rows that carry it
(`hub/domain/observability/egress/rows.py:76`), and the runner stops reading it.

## Where the runner counts

**The rolling window.** `UsageTotals` (`runner/usage/repository.py:171`) gains `estimated_cost_usd` and a `spend_usd` of
the two together, and its `cost_partial` takes the board's meaning; `reaches(cap)` compares `spend_usd`. `usage_since`
(`runner/store/internal/usage_store.py:45`) sums `usage_facts.estimated_cost_usd` (`runner/store/schema.py:564`) beside
`cost_usd`, and counts a row partial only when both are null (`:53`). The ceiling's `SpendCeiling`
(`runner/loop/steps.py:150`–`151`) and the budget lever ([levers.md](./levers.md)) read the same `UsageTotals`, each
over its own window.

**The chunk.** The chunk cap already reads the hub-derived total rather than a local sum (`_spend_cap_breach`,
`runner/lifecycle/drain.py:201`), and that total already carries both amounts and both partial readings (`ChunkSpend`,
`runner/node_steps/chunk_state.py:24`). `spend_cap_reached` (`runner/lifecycle/model.py:731`) compares `cost_usd` plus
`estimated_cost_usd`, and `spend_cap_partial_note` (`:737`) and the log line at `drain.py:221` read `cost_partial`
instead of `billed_partial`.

**The words.** One rendering serves every surface, so the runner's reasons read as the board does: `foundation/` gains
the spend figure's rendering — one amount to the cent, a leading `~` when it carries an estimate, a trailing `+` when it
is PARTIAL — which both daemons evaluate identically (`bzh:shared-kernel`). The hub CLI's `Cost.rendered`
(`hub/cli/views.py:38`) moves onto it, and so do `spend_ceiling_reason` (`runner/throttle/pause.py:193`), which today
appends `(PARTIAL — true spend may be higher)` for a billed shortfall, and `spend_cap_detail`
(`runner/lifecycle/model.py:742`). A ceiling engagement then reads
`spend ceiling $60.00 reached over the trailing 24h (spend ~$61.37+)`.

The prose that states the old reading is rewritten with the code: `invocation_cost`'s docstring
(`runner/harness/usage.py:83`–`85`), the comment on `UsageSample.estimated_cost_usd` that an estimate is never capped
(`:39`), and `spend_cap_reached`'s docstring.

## What the operator reads

`docs/deployment/spend.md` changes in four places. §The two caps keeps both breakers and their manual clear, and says
they trip on billed plus estimated spend. §Partial totals states the one PARTIAL. §Estimated cost loses its closing
paragraph — that an estimate never feeds either cap, and that the caps stay blind to subscription spend — and says
instead that every line reads the figure with its `~`. A new §Throttles, ahead of §The two caps, introduces the throttle
and breaker layers, the `[runner.throttle]` table and its levers ([configuration.md](./configuration.md)), and the
budget set below the ceiling it stands beside.

A runner whose spend is mostly estimated reaches its existing ceiling and chunk cap sooner than it did, since the caps
now see what they were blind to. That is the change the plan asks for, and [carry-over.md](./carry-over.md) names it for
the operator before it lands.
