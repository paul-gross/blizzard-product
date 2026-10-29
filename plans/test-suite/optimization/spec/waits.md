# Waits that end when the thing has happened

A fixed wait is either too long, and every run pays for it, or too short, and the test is flaky on a slow machine. The
fixed waits outside the service tier that this epic owns are three, in two files, and each is replaced by a wait on the
event it stood for. E2E's tick spacing, its `poll_until` intervals, and its termination allowance move with their
scenarios under `epic:test-suite/fleet-tier`, and the crash sweep's are untouched.

## `tests/test_cli.py`

`_serve_on` stands up a fake runner daemon on a Unix socket, and when its reply is `None` it holds the connection with
`time.sleep(LOCAL_CLIENT_TIMEOUT * 3)`, a daemon that accepts and never answers. That sleep runs on a daemon thread and
costs nothing. What costs is the client side: `test_an_ambiguous_liveness_answer_fails_closed[None-did not answer]`
waits out the production client's `LOCAL_CLIENT_TIMEOUT`, five seconds (`src/blizzard/runner/cli/daemon.py`), before the
liveness check gives up.

The test's subject is that silence fails closed, not that the timeout is five seconds. `daemon_holding` takes the client
timeout as a keyword, defaulting to `LOCAL_CLIENT_TIMEOUT`, and the silent case passes a quarter of a second. The value
of the production constant is asserted where it is defined, if anywhere, not by every test that meets it. The fake
daemon's hold becomes an `Event` the fixture sets on teardown, so the thread ends with the test rather than fifteen
seconds later.

## `tests/test_runner_harness_opencode_diagnostic.py`

`test_timeout_kills_and_reaps_an_opencode_descendant` and
`test_successful_run_rejects_and_reaps_a_pipe_closing_descendant` each start a launcher whose descendant sleeps half a
second and then writes a marker, and the test sleeps `0.7` seconds before asserting the marker is absent. The absence is
a stand-in for the claim the tests make, that the descendant was reaped, and it costs 0.7 seconds each while proving it
only as long as the machine is fast enough to have written the marker by then.

Each descendant writes its pid to a file as its first act, and the test asserts the reaping directly: once `run`
returns, the recorded process is gone, checked with `os.kill(pid, 0)` raising `ProcessLookupError` and, where the
descendant is the test's own child, `os.waitpid` reporting it. The check polls with a bound of five seconds, as the
file's other bounds are, and returns the moment it holds. The marker and the half-second sleep go, since the descendant
no longer has to live long enough to be caught misbehaving.

## The rule going forward

`blizzard-context`'s `verification/blizzard/tier-rules.md` already asks the service tier for waits that end when the
system speaks, through the foundation's companion. This epic extends the same rule, `bzh:service-waits-on-the-system`,
to every tier by name, in the change that lands the last of the three: a test waits on an event, a state, or a process
it can observe, and a bounded poll is its upper limit, never its duration.
