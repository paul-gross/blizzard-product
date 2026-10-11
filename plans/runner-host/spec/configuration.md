# Host configuration contract

## The file and the directories

A host is a directory holding `blizzard-host.toml`. Each runner on it keeps a directory of its own — its store, its
published harness bundle, and its scratch directories (`worker-stdout/`, `elicitation-output/`, `worker-tmp/`) — named
by the runner's `dir`, relative to the host directory or absolute. A runner's directory is exactly what a runner's root
is today, minus `blizzard-runner.toml`, which is how existing runners move under a host without moving a file
([carry-over.md](./carry-over.md)).

`RunnerConfig.load` (`runner/config.py:1050`) gains a sibling, `HostConfig.load`, that reads `blizzard-host.toml` and
projects one `RunnerConfig` per `[[runner]]` entry, with the host's and the bound workspaces' keys filled in. Everything
downstream of a `RunnerConfig` keeps reading the shape it reads today, apart from the agent limit, which moves to the
binding ([workspaces.md](./workspaces.md) §Parallel agents). `blizzard runner host` refuses a directory holding a
`blizzard-runner.toml` and no `blizzard-host.toml`, naming the carry-over procedure ([carry-over.md](./carry-over.md)).

```toml
hub_url = "https://blizzard.grosscode.net"
host = "127.0.0.1"
port = 8431
public_url = ["http://127.0.0.1:8431", "https://xps.example.ts.net:8431"]
trusted_proxies = ["127.0.0.1"]

[auth]
superuser = "paul"
hub_role_default = "mirror"

[[subscription]]
slug = "anthropic"
name = "Anthropic"
provider = "anthropic"

[[subscription]]
slug = "openai"
name = "OpenAI"
provider = "openai"

# One checkout, declared as two workspaces so each runner keeps environments of its own.
[[workspace]]
id = "blizzard-claude"
root = "/home/pgross/projects/blizzard-blizzard/blizzard-workspace"
envs = ["r1", "r2", "r3", "r4", "r5", "r6"]
base_branch = "master"
prompt_package = "winter"

[[workspace]]
id = "blizzard-openai"
root = "/home/pgross/projects/blizzard-blizzard/blizzard-workspace"
envs = ["oce1", "oce2", "oce3", "oce4"]
base_branch = "master"
prompt_package = "winter"

[[runner]]
name = "r-claude"
dir = "../runner"
subscription = "anthropic"
projects = [{ project = "blizzard", workspace = "blizzard-claude", agents = 2 }]

[runner.claude_code]
enabled = true

[runner.opencode]
enabled = false

[[runner]]
name = "r-chatgpt"
dir = "../runner-opencode"
subscription = "openai"
projects = [{ project = "blizzard", workspace = "blizzard-openai", agents = 1 }]

[runner.claude_code]
enabled = false
```

## Where each key lives

Every key `RunnerConfig` reads today has exactly one home.

| Level               | Keys                                                                                                                                                                                                                                                                           |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Host                | `hub_url`, `host`, `port`, `public_url`, `trusted_proxies`, `session_secret_env`, `[auth]`, `[tracing]`, `[transcripts]`, `[worker_stdout]`, `host_token`, `[hub_tokens]`, `[[subscription]]`, `[[workspace]]`                                                                 |
| `[[workspace]]`     | `id` (today's `workspace_id`), `root` (`workspace_root`), `provider` (`workspace_provider`), `envs` (`workspace_envs`), `max_environments`, `[[workspace.repo]]` (`[[workspace_repo]]`), `base_branch`, `prompt` / `prompt_file` / `prompt_package` (`workspace_prompt*`)      |
| `[[runner]]`        | `name`, `dir`, `subscription`, `projects`, `gates`, `runner_prompt*`, `transcripts_root`, `[harness]` (`autonomy`, `config_dir`), `[claude_code]` and its legacy root keys, `[opencode]`, `[models.aliases]`, `[effort.aliases]`, `[cost]`, `[queue]`, `[context]`, `[worker]` |
| `runner.projects[]` | `project`, `workspace`, `agents` (today's `max_agents`)                                                                                                                                                                                                                        |

Worker security is the runner's: `[harness]` autonomy and bundle, `[worker]`'s spawn-environment allowlist, and the
Claude Code legacy keys (`harness_permission_mode`, `worker_settings_path`) all sit under `[[runner]]`, so two runners
on one host may run at different autonomy levels.

`db_url` is not configurable under a host: each runner's store is `sqlite:///<dir>/data/runner.db`, and the host's own
store is `sqlite:///<host dir>/data/host.db` ([subscriptions.md](./subscriptions.md) §The host store).

## Workspaces

A workspace is a root and the environments it names within it. Two workspaces may share a root, as long as no
environment name appears in both; that is how one checkout gives each runner a set of its own. One workspace naming
every environment pools them instead. Both are valid, and the host treats each workspace's environments as one pool
whichever is chosen ([workspaces.md](./workspaces.md) §The shared pool).

## Projects

`[[runner]].projects` is a list of bindings, each `{ project, workspace, agents }`: a project id or slug, as today's
`projects` list takes; a `[[workspace]].id` of this host; and the count of agents that may work the binding at once, a
positive integer.

- **A list** declares exactly its projects, in order, binds each to its workspace, and caps each at its `agents`.
- **`[]`** serves nothing, as today.
- **Absent** is not accepted: every hosted runner states what it serves.

## Hub credentials

The host holds two kinds of credential ([hub.md](./hub.md) §The host credential).

The **host token** authenticates the host itself: its reports, its subscription usage, panel sign-in, and adding or
retiring its runners. The host resolves the first of the environment variable `BZ_HOST_TOKEN`, the host file's
`host_token` key, and `BZ_HOST_TOKEN` in the host directory's `.env`. The host's id is not configured; the hub answers
with it on the host's first report, and the host keeps it in its store (`host_identity`,
[subscriptions.md](./subscriptions.md) §The host store).

A **runner token** identifies one runner to the hub, which knows a runner's work by its token alone
(`wire/runner.py:53`). Tokens are per runner, keyed by the runner's `name`. For each runner the host resolves the first
of:

1. The environment variable `BZ_HUB_TOKEN_<NAME>`, where `<NAME>` is the runner's name upper-cased with every character
   outside `[A-Z0-9]` replaced by `_` — `r-claude` reads `BZ_HUB_TOKEN_R_CLAUDE`, `r-chatgpt` reads
   `BZ_HUB_TOKEN_R_CHATGPT`.
2. The host file's `[hub_tokens]` table, keyed by runner name: `[hub_tokens] r-claude = "…"`.
3. `BZ_HUB_TOKEN` in the runner directory's `.env` (`hub/token_file.py:89`), which is where `blizzard runner add` writes
   a new runner's token, and where a runner moved under a host already keeps its own.

The unsuffixed `BZ_HUB_TOKEN` in the process environment is never read under a host; it names no runner. A host whose
file carries `host_token` or `[hub_tokens]` warns at startup when the file is readable by anyone but its owner. The
panel's session secret is the host's, read from the process environment under `session_secret_env` as today
(`runner/config.py:105`).

## Validation

`HostConfig.load` refuses the file, naming the offending entry, when:

- two `[[runner]]` entries share a `name` or a `dir`, two share the same token environment-variable name after
  normalisation, two `[[workspace]]` entries share an `id`, or two `[[subscription]]` entries share a `slug`;
- a runner names a `subscription` the host does not declare, or a binding names a `workspace` the host does not declare;
- a runner binds the same project entry twice, whatever the workspaces;
- a binding's `agents` is absent or not a positive integer;
- two workspaces on the same root name the same environment;
- a runner enables no harness (`harness/wiring.py:56`), as today.

A runner may leave `subscription` out: it spends no declared plan, reports no roster, and a usage limit brakes it alone,
with no fallback reset time ([subscriptions.md](./subscriptions.md) §The brake).

Two bindings naming the same project by different spellings — its slug and its id — pass the file check and are caught
at registration, where both resolve to one project id: the runner refuses to claim until the configuration is fixed, and
its status names both entries.
