# Runner harness configuration contract

## Inputs and ownership

`blizzard-runner.toml` owns two runner-wide inputs: an `autonomy` value and an optional path to an operator-owned
harness configuration bundle. The hub, graph, and chunk cannot override either. A proposed shape, using one setting for
every harness this runner launches, is:

```toml
[harness]
autonomy = "auto" # normal | auto | dangerous
config_dir = "/etc/blizzard/harness-config"
```

The bundle has one directory per harness, `claude-code/` and `opencode/`. For example:

```text
harness-config/
  claude-code/
    settings.json
    mcp.json
    agents.json
    plugins/
  opencode/
    opencode.json
    plugins/
```

The harness directory names are the bundle contract; the files within them follow that harness's native conventions. The
paths above illustrate settings and companion material, not an exhaustive or interchangeable file schema. An operator
may leave a harness directory out. The harness binding declares which entry points it consumes from its directory; an
unrecognized top-level entry point cannot quietly be ignored. Native companion files referenced by a supported entry
point come along with it. Blizzard does not translate their contents into a parallel schema or promise that every
feature fits in one JSON document.

## From bundle to worker process

The directory is an input to the runner, not something the harness discovers automatically:

1. At `runner host` startup, after loading `blizzard-runner.toml` and before accepting work, resolve `config_dir`, read
   the two harness directories present, and validate the entry points each binding supports. The one-shot `runner tick`
   path uses the same loader. A missing, malformed, or unsupported configured entry point fails with its source path and
   cause before a worker launches.
2. Materialize a private, runner-owned effective directory under the runner runtime root. Copy the supported native
   material so the worker sees one startup snapshot, preserve relative file references, and compose native settings with
   blizzard's mandatory hooks, plugin, and headless rules. Do not edit the operator's directory or any project worktree.
   Validate collisions and publish the new effective directory only when the entire composition succeeds.
3. Give each harness binding the paths to its effective native files and the resolved `autonomy` value. Each binding
   delivers the material through that harness's actual configuration entry points:
   - Claude Code: pass the composed settings via `--settings` on every unattended invocation; pass optional bundled MCP,
     agent, and plugin inputs via `--mcp-config`, `--agents`, and `--plugin-dir`. A settings file alone does not load
     these companions.
   - OpenCode: pass the composed JSON via `OPENCODE_CONFIG` and identical serialized contents via
     `OPENCODE_CONFIG_CONTENT`. Point `OPENCODE_CONFIG_DIR` at the effective directory for native plugin and other
     config-directory assets. `OPENCODE_CONFIG` alone does not load the rest of the bundle.
4. Apply the bound arguments and environment to fresh, resumed, nudged, and judged workers; a resumed session does not
   recover invocation flags or child environment from the original launch. Subagents must see the same effective
   configuration. Make the resolved source paths, effective paths, and chosen autonomy inspectable without printing
   config contents that may contain secrets.

Changes to operator files require a runner restart; in-flight turns keep the snapshot they started with. Runner
initialization (`blizzard runner init`) may regenerate runner-owned settings and plugin files, but never edits or
replaces the operator's bundle. With no bundle, existing generated configuration remains the effective input.

Loading anything twice is never the goal; it is a hazard of OpenCode reading the same composition through three entry
points. A plugin both named in the composed JSON and present under `OPENCODE_CONFIG_DIR` could register twice — two
heartbeats per tool call. Delivering OpenCode's configuration includes proving each plugin and setting loads exactly
once; if the loader duplicates one, use only the entry points needed while still making companion files available.

## Existing local settings

The bundle is an additional runner-supplied source, not a switch that disables a worker's existing harness
configuration. User settings on the runner and settings in the worker's project still load by default. Unrelated local
settings remain available; for an overlapping ordinary setting, the composed runner/bundle value wins over user and
project values. Harness-managed settings still take their native higher precedence. This is a per-key rule, not a
promise that an entire local file disappears.

Claude Code places `--settings` above user, shared-project, and project-local settings, but below managed settings.
Lists such as permission rules may combine across scopes rather than replace one another; project or user denials are
not turned into approvals merely because the bundle allows a tool. OpenCode merges global, project, and custom config;
`OPENCODE_CONFIG` alone is below project config, while `OPENCODE_CONFIG_CONTENT` is applied after ordinary project and
config-directory sources, below managed config. That second delivery is why the composed JSON, rather than an uncomposed
operator file, must be supplied as content. Native directory assets such as plugins also remain discoverable from the
project and user scopes. The binding must detect or explicitly resolve duplicate plugin identities and colliding native
entries, rather than quietly loading both copies or claiming its chosen value won without proof.

Runner-required hooks and headless rules have a separate invariant: neither the operator bundle nor ambient local
settings may silently remove them. If a harness's merge or managed-policy behavior prevents that guarantee, report the
binding as unavailable with the conflicting source instead of claiming the runner's rule won. Prove precedence with real
resolved-config output and a tool call, not only by inspecting the files blizzard wrote.

## Autonomy

`autonomy` is a single runner setting, not one per harness. It has exactly three values, represented in configuration as
`normal`, `auto`, and `dangerous` (`Autonomy.Normal`, `Autonomy.Auto`, `Autonomy.Dangerous` in code). The binding
translates the value for every unattended worker invocation, including a fresh session, resume, nudge, and judgement:

| Runner value | Intent                                       | Claude Code                           | OpenCode      |
| ------------ | -------------------------------------------- | ------------------------------------- | ------------- |
| `normal`     | Ordinary approval behavior; no auto-approval | `--permission-mode manual`            | Omit `--auto` |
| `auto`       | Harness-native automatic approval            | `--permission-mode auto`              | `--auto`      |
| `dangerous`  | Approve anything the harness can approve     | `--permission-mode bypassPermissions` | `--auto`      |

OpenCode's `--auto` is its broadest approval mode, so `auto` and `dangerous` have the same OpenCode launch behavior.
This is an intentional many-to-one mapping, not a claim that OpenCode bypasses explicit denies the way Claude Code may.

`normal` means the operator grants explicitly and everything else is refused immediately. For headless Claude Code
invocations, `--permission-prompts none` makes any request still requiring a person a definite denial, rather than a
prompt awaiting a nonexistent host. OpenCode gets the same outcome from its composed configuration rather than from
whatever its headless run happens to do with a pending request: in `normal`, every permission that would resolve to
`ask` — written by the operator, by local settings, or left to OpenCode's own default — is composed as `deny`. In both
harnesses the refusal returns to the worker as an ordinary tool error, and the worker takes another route; the runner
does not intervene. For an absent `autonomy` setting, existing installations retain today's effective default: Claude
Code `bypassPermissions`, OpenCode `--auto`. The runner's current `harness_permission_mode` setting for Claude Code must
have one unambiguous migration path: while a config still uses it without `autonomy`, preserve its existing behavior; if
both are supplied, report the conflict and require the operator to choose the single runner-wide setting. A new config
scaffolds `autonomy = "dangerous"` to preserve today's defaults.

Autonomy decides how the harness handles approval requests; it does not remove native tool rules from the operator's
bundle. `Dangerous` means the broadest native approval, not unrestricted host access: OpenCode `--auto` still refuses
tools explicitly marked `deny`. Whether Claude Code's `bypassPermissions` honors the operator's own deny rules is the
operator's concern, not blizzard's: the operator configured the harness and chose the mode, so blizzard neither
compensates nor claims those denies are enforced in `Dangerous`, and the operator documentation says so. Runner-owned
headless denials are different — blizzard depends on them — and must stay effective in every mode through a
harness-native mechanism or an equivalent runner-owned outcome. Today's default already pairs `bypassPermissions` and
`--auto` with those denials; a regression test holds that for every autonomy value. Apply the same scrutiny to user or
project settings that might weaken runner-owned requirements.

In a headless run of any mode, a tool request that still needs a person cannot remain pending indefinitely: it resolves
as a definite denial, tested for both CLIs. An interactive takeover can present prompts to its operator; the binding
states separately which unattended flags, if any, its attended command retains.

## Effective configuration

The effective config is a composition of the operator's native bundle and the runner's mandatory worker wiring. Claude
Code's heartbeat and session-end hooks and its runner-owned denials remain; OpenCode's heartbeat/identity plugin and
unattended `question` denial remain. Compatible operator hooks, plugins, and permissions are additive, retaining native
ordering where it matters; unrelated settings pass through. A collision with mandatory wiring or an unsupported
combination fails with the native setting's path. The operator can inspect what the next worker will load without
editing generated files. The runner never writes harness config into a project worktree, and no hub-controlled input
becomes a path to local harness config.

Each harness binding owns its own mandatory denials; there is no shared, harness-neutral denial list. Claude Code's
denials — `ScheduleWakeup`, `Monitor`, the `Cron*` tools, `RemoteTrigger`, and `EndConversation` — exist because those
tools defer work to a later turn a headless Claude Code session never gets, or end it early; they name Claude Code tools
and mean nothing to another harness. OpenCode's `question` denial exists because no one is attached to answer it. The
Claude Code list moves out of the shared harness module into the Claude Code binding, beside the OpenCode binding's own
document, so each composition adds only its own harness's denials and a new binding declares its own rather than
inheriting another's.

For each harness, prove with the real CLI that the effective config and its companion files load alongside the runner's
mandatory wiring, and that user/project config, resumes, judgements, and subagents do not silently replace either side.
The proof includes a tool denied by the operator in a mode that honors native denials, an operator-added MCP or plugin,
a runner heartbeat, and each autonomy value. Where a native source has different precedence or composition semantics,
the binding documents and enforces its own route rather than assuming the other harness's behavior.
