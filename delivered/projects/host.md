---
epic: projects/host
refinement: pristine
status: retired
---

# Plan — `epic:projects/host`

## The machine

Three projects on a laptop mean one runner host, not three runners in three directories, and that promise is kept on the
machine by `epic:runner-host`, not here. Its host binds each runner's projects to the workspace that serves them, so a
claim lands in the one workspace its chunk's project is bound to. This epic's `host` slice is retired unbuilt: its work
is `epic:runner-host`'s.
