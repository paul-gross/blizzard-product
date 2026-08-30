# OpenCode compatibility specification

The compatibility proof runs as the planned `blizzard:manual-opencode-compatibility` verification method against one
pinned OpenCode version and records machine-readable fixtures for every external shape the production binding parses.
The compatibility slice adds that method to Blizzard's verifiability matrix and gives it a runner diagnostic command
before production behavior depends on it. A version is compatible only when the required rows pass together; a result
from documentation alone is not evidence.

| Probe                   | Evidence required                                                                                                                                                                                                                        |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fresh headless turn     | `opencode run --format json` starts one bounded process, creates one root session, carries the root `sessionID` on its first emitted record, performs an edit, and exits when the turn completes                                         |
| Resume                  | `opencode run --session <id>` continues the same conversation and exits independently of the earlier process                                                                                                                             |
| Process control         | The recorded PID remains alive for the duration of a long tool call, responds to termination, and does not leave a second process able to keep mutating the worktree                                                                     |
| Judgement               | A resumed synchronous turn returns final assistant text from which `<Choice>` and the assessment can be isolated without including tool output                                                                                           |
| Usage                   | The event stream or returned message identifies model, input, output, reasoning, cache read, cache write, and cost where the provider reports them                                                                                       |
| Root lifecycle hook     | A runner-owned plugin observes tool completion and errors for the root session, and a plugin that fails to load degrades heartbeat without affecting the turn's outcome                                                                  |
| Permission policy       | Non-interactive execution neither waits for a person nor bypasses an explicit deny, and the worker can still invoke the Blizzard ask command                                                                                             |
| Model and effort        | `provider/model` selection and the provider's `max` variant survive fresh mint, resume, and judgement for the intended ChatGPT 5.6 Luna run                                                                                              |
| Takeover                | The TUI can enter the recorded session in the recorded workdir and continue it under attended permissions                                                                                                                                |
| Transcript read         | `opencode export <session-id>` returns the session's messages and parts **during** a live turn as well as after exit, without disturbing the writer                                                                                      |
| Transcript cursor       | Message and part ids are unique and stable across repeated live exports, preserve their order as a tool moves from pending to complete, and let records appended after compaction be distinguished without re-admitting retained history |
| Child sessions          | A child can be enumerated from its root and any stable relationship to the spawning tool call is identified                                                                                                                              |
| Configuration isolation | Runner-owned plugin and permission configuration load for the worker without being written into the project repository or leaking one lease's identity into another                                                                      |

The proof retains representative success, provider error, permission denial, interrupted tool, compaction, and
child-session fixtures. Parsers test against those fixtures and reject unknown required shapes explicitly. A harness
upgrade reruns the proof before its observed version may become available to acquisition.

The diagnostic runs in a disposable git repository against the operator's ordinary OpenCode authentication. It accepts
the harness binary, model, and variant explicitly, prints the observed version and one result per probe, and writes
sanitized fixtures to a caller-selected output directory. It never prints, copies, or persists provider credentials. The
live ChatGPT probe is opt-in because it spends real tokens; every parser and orchestration test consuming its shapes
remains hermetic afterward.

Three outcomes are valid:

- **Supported** — production behavior may depend on the observed contract and carries a regression fixture.
- **Degraded** — the adapter returns the harness-neutral absence already defined by the seam, and the operator-facing
  capability says what is unavailable.
- **Blocking** — the execution or transcript slice depending on the behavior does not ship for that version.

Three of these are already answered against OpenCode 1.18.25 and enter the proof as regression fixtures rather than open
questions: every `--format json` record carries the root `sessionID` from the first one onward, `step_finish` carries
the full token breakdown, and `opencode export <session-id>` returns `{info, messages[{info, parts}]}`. What the proof
still has to settle is per-turn PID ownership, whether an export taken mid-turn is coherent, whether message and part
identities remain stable through live updates and compaction, and whether a subscription-authenticated `cost` is ever
anything but zero.

The proof must settle those, along with permission behavior under `--auto`, before the execution implementation is
accepted. These are boundaries the supervisor treats as truth; ambiguity at any one of them becomes a crash-correctness
defect rather than a cosmetic incompatibility. The matrix entry and its detailed manual-method spoke are companion
changes of the diagnostic, so the release gate always resolves to a declared verification method rather than prose in
this plan alone.
