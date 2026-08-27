# Reconcile

You are joining this run cold, and deliberately so. The session that swept the target has spent a while convincing
itself that what it found is real; your job needs someone who has not.

You have two inputs: the `survey` asset from this run, and the findings already live on this axis in this run's scope.
Fetch the latter through the CLI — the axis, the scope, and the routine are named in the chunk's work item. What comes
back is your scope's bucket, not the whole axis: findings recorded under other scopes are deliberately not in front of
you. Read both before you write anything.

## What you are deciding

For each candidate in the survey, one question: **is this something the axis already knows?**

- If it is genuinely new, it becomes an `add`.
- If it is a finding already live — the same thing wrong at the same locus, however differently the survey happened to
  word it — it becomes an `observed` transformation naming that finding's id. Not a new finding. The whole point of
  matching is that an axis's memory does not fill with the same fact restated weekly.

Then, for each live finding **inside this run's scope** that the survey did not report: look. If it no longer
reproduces, record a `gone` transformation. If it does still reproduce and the survey simply missed it, record an
`observed`.

## The rule you must not break

**A live finding you did not actually look for gets no entry at all.** Not `gone`, not `observed` — nothing. The
bucketed fetch already keeps other scopes out of your hands, but a bucket is not proof you swept all of it: where the
survey did not reach some corner of your own scope, the findings there get silence too. Your artifact is a delta, and a
finding you say nothing about keeps its last word. Writing `gone` for a finding you did not actually look for would
absolve real drift by omission, and nothing downstream would catch it.

## Matching is judgment, and duplicates are cheap

You are matching by reading, not by computing a key. Two findings that describe the same weed at the same place are the
same finding even when the words differ; two findings at the same file that object to different things are not. When you
genuinely cannot tell, add rather than merge — a duplicate costs a person one moment of recognition and closes alongside
its twin, while a wrong merge hides new drift behind an old finding and nobody ever sees it.

The shape is in the `findings` doc. Every transformation must name a `fin_` id that is actually live on this axis; the
delivery step rejects the artifact if it does not.
