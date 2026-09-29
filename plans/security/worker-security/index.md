# Plan — `epic:security`, worker-security slice

An operator can tune a coding harness for their own work, but a worker launched by blizzard inherits settings the runner
generates. Editing those files by hand does not last through `blizzard runner init`. The operator needs to bring another
set of harness configurations into the runner at startup — permissions, MCP servers, plugins, and other native settings
— without losing the hooks and headless behavior blizzard needs to do its job.

The runner-auth slice of this epic — runners signing in to the hub with hub-minted tokens — is already delivered. This
slice is about how the operator configures the coding harnesses those runners launch.

## What to build

**Bring an operator-owned configuration bundle to the runner.** One runner-local path points to a set of harness-native
files, grouped by harness. The operator can keep settings, MCP definitions, plugin files, and other supported companion
files there rather than editing blizzard's generated files or scattering these choices through a blizzard-specific
schema. The runner loads that bundle when it starts; it never rewrites the source. An absent bundle preserves current
behavior. This choice belongs to the runner, not the hub or an individual chunk.

**Compose what each worker actually receives.** The runner builds an inspectable effective configuration from the bundle
and its own required additions. Its heartbeat and session-end hooks, OpenCode plugin, and denials of tools a headless
worker cannot use must survive; the operator's unrelated native settings must survive too. Combine compatible additions
to hooks, plugins, and permission rules rather than replacing whole sections. A conflict names the offending setting and
fails at runner startup instead of silently overriding either side. `blizzard runner init` may refresh generated files
but must leave the operator's bundle intact.

Those required denials belong to the harness that needs them. Claude Code's exist so a session does not schedule itself
a later turn it will never get, or end early; OpenCode's exist so it does not ask a question no one is there to answer.
Each harness's interface carries its own list, so an OpenCode worker never receives Claude Code's rules and the next
harness declares its own rather than inheriting another's.

Existing user and project settings still load. The runner-supplied configuration takes precedence for overlapping
ordinary settings, while unrelated local settings remain. Native permission rules can combine, and managed settings
retain the harness's own precedence; a clash that prevents blizzard's required worker behavior must be visible rather
than silently ignored.

**Give the runner one autonomy choice.** `Normal` uses the harness's ordinary approval behavior, `Auto` uses its auto
mode, and `Dangerous` grants the broadest available approval. Claude Code has distinct modes for all three; OpenCode
uses `--auto` for both `Auto` and `Dangerous` because that is its broadest mode. A runner without the setting keeps its
current behavior. This choice governs approval, while the operator's native configuration still carries tool rules.

**Make normal mode usable without a silent stall.** A normal permission prompt in a headless run must resolve by an
explicit route, be denied, or surface as an escalation; it cannot wait forever for a person who is not attached. Fresh
sessions, resumes, judgements, and their subagents must get the intended configuration. Document the different behavior
of interactive takeover, where an operator can answer prompts.

An operator should be able to deny a native tool, add a harness extension, restart the runner, and see both choices in
the next worker without losing blizzard's heartbeat. Tool permissions govern harness behavior, not the authority of
every command a shell can execute. Workers still need to fetch, rebase, commit, and push feature branches; protecting
the default branch is a forge rule, and a rebase may require `--force-with-lease`.

The [specifications](./spec/index.md) own the bundle layout, autonomy mapping, composition, and invocation rules. They
are the implementation entry point for this slice.
