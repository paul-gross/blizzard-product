# OpenCode conversation parity

An operator investigating an OpenCode worker should not lose the transcript, context warning, or cost recovery they rely
on simply because the runner selected a different acceptable harness. OpenCode conversations enter the transcript lane
through their own source and normalizer, then become the same normalized turns the runner panel, hub archive, and
analytics already consume.

This slice reads the root conversation while it is active and after it exits, follows child sessions as sidechains where
OpenCode exposes a stable relationship, and carries honest truncation or an unlinked child where it does not. It
supplies the context-size and transcript-size measurements used by session rotation only when OpenCode's values have the
same meaning the policy expects. A measurement that cannot be derived faithfully remains unknown, because an invented
zero would quietly disable a bound while claiming it was enforced.

Completed invocations normally record usage from their event stream. The transcript source also supplies the recovery
path for a process killed before its final event survived, preserving token attribution without estimating dollar cost
that OpenCode did not report.

The established transcript lane remains canonical in `epic:transcripts`; this slice adds the OpenCode source that feeds
it. The [OpenCode transcript specification](./spec/transcripts.md) owns that binding.
