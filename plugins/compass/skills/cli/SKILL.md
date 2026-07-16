---
name: cli
description: >-
  Operate the Compass platform from the command line with the `compass` CLI:
  run and follow executions, list/inspect workflows, agents, executions,
  connections, providers, projects, conversations, and approvals. Use when the
  user wants to run/execute a Compass workflow or agent, deploy or check
  something on Compass, tail an execution, list or inspect any Compass resource,
  approve/reject a human-in-the-loop request, or switch project/profile.
allowed-tools: "Bash(compass:*)"
---

# Operating Compass with the `compass` CLI

The `compass` CLI is a thin, machine-friendly wrapper over the Compass SDK. Every
command opens a client, makes one call, and prints the result. This skill is how
you (Claude) drive it correctly.

## First: ground yourself against the installed CLI

The CLI evolves independently of this skill, so never trust a memorized flag —
confirm against what is actually installed.

1. Check it exists and its version:
   ```
   compass --version
   ```
   If the command is not found, the user installs it from their Compass platform —
   the wheel is served per-org, gated by their login:
   - Go to **Settings → Developer → SDK & CLI** (`/settings/sdk`) and click
     **Download wheel** (the page also shows the exact version-stamped command).
   - Install the downloaded wheel with its `[cli]` extra:
     ```
     uv tool install "./compass_core-<version>-py3-none-any.whl[cli]"
     ```
     (`pipx install "./compass_core-<version>-py3-none-any.whl[cli]"` works too.)
   - Authenticate: `compass auth login --url <your-compass-url>` (or set the env
     vars below).

   Do not proceed until `compass --version` works.

2. Discover exact flags for any command group before relying on them:
   ```
   compass --help
   compass <group> --help          # e.g. compass workflows --help
   compass <group> <cmd> --help    # e.g. compass workflows create --help
   ```
   Prefer `--help` output over anything written here when they disagree.

## Auth: prefer env vars, they bypass interactive login

The CLI resolves config as **flag → env var → stored profile**. For non-interactive
use, environment variables are the cleanest path and fully bypass `compass auth login`:

| Variable | Purpose |
|----------|---------|
| `COMPASS_API_KEY` | Org-scoped API key (`ak_…`). Required. |
| `COMPASS_BASE_URL` | Platform base URL. Required. |
| `COMPASS_PROJECT_ID` | Active project (sent as `X-Compass-Project-Id`). Optional. |
| `COMPASS_PROFILE` | Named profile to use from the config file. Optional. |

Check whether the user is authenticated before running real commands:
```
compass auth whoami
```
If not authenticated, tell the user to either export `COMPASS_API_KEY` +
`COMPASS_BASE_URL`, or run `compass auth login` themselves (it's interactive —
you cannot complete it for them). The stored config lives at
`~/.config/compass/config.json` (chmod 0600).

**Never** print, echo, or write the API key into files or command output.

## Always request JSON output

Add the per-command `--json` flag on read commands so you can parse results
reliably instead of scraping the table view:
```
compass workflows list --json
```
The global `-o/--output {table,json,yaml}` flag also exists but must come
**before** the subcommand (`compass -o yaml workflows list`) — trailing
`-o json` is a parse error. Prefer `--json`. Other global flags (also
before the subcommand): `--profile`, `--base-url`, `--api-key`, `--project`
(per-command project override).

## Command groups

Confirm subcommands and flags with `--help`; this is the map, not the contract:

| Group | What it does |
|-------|--------------|
| `compass execute` | Dispatch an execution of a deployment (`-d`) or workflow (`-w`) with inputs (`-i`), optionally following live output (`-f`). The main "run something" command. |
| `compass workflows` | `list`, `get`, `create`, `update`, `validate`, `deploy`, `delete` workflow definitions. See the **build-workflow** skill for authoring. |
| `compass nodes` | `list` — the FULL placeable palette in one call: components + the agent kind + the project's integrations, each row tagged `node_type`. The ground truth for what this deployment supports. |
| `compass components` | `list`, `get <type>` — the component catalog in depth (config fields + input/output schemas). Read-only. |
| `compass agents` | `list`, `get`, `create`, `update`, `execute`, `delete` agents. See the **build-agent** skill for authoring. |
| `compass executions` | `list`, `get`, `node-executions`, `tree` — inspect past/running executions (the observability surface). |
| `compass approvals` | `list`, `get`, `approve`, `reject` — human-in-the-loop gates. |
| `compass connections` / `providers` / `gateways` | CRUD for connections (model/provider/tool credentials). `connections` also has `health`. |
| `compass integrations` | Custom OpenAPI connectors: `list`, `get`, `register`, `delete`. |
| `compass projects` | `list`, `use <project-id>` — switch the active project. |
| `compass conversations` | `list`, `get`, `create`, `runs`, `delete`. |
| `compass auth` / `compass config` | `login`/`logout`/`whoami`; `list`/`path`/`use <profile>`. |

## Common recipes

Run a deployed workflow and follow it live:
```
compass execute -d <deployment-id> -i topic="Q3 report" -f
```

Run a draft workflow by id:
```
compass execute -w <workflow-id> -i key=value --json
```

Inspect what happened in an execution:
```
compass executions get <execution-id> --json
compass executions tree <execution-id> --json
```

Handle a pending approval:
```
compass approvals list --json
compass approvals approve <approval-id>
```

## Rules

- Confirm flags with `--help` before running anything you're unsure of.
- Use `--json` whenever you need to read a result (trailing `-o json` does not parse).
- For inputs, the syntax is `-i key=value`, repeatable, or `-i @file.json` for a
  JSON body — but verify with `compass execute --help`.
- Destructive commands (`delete`, `reject`) change real state — confirm intent
  with the user first and echo back exactly what will be affected.
- Never expose the API key.
