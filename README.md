# Sage Compass — Claude Code Marketplace

Claude Code plugins for the [Sage Compass](https://sageai.solutions) AI enablement
platform. Install the `compass` plugin to let Claude help you **build workflows and
agents** and **operate the `compass` CLI** directly from your editor or terminal.

## Install

```bash
# Add this marketplace (one time)
/plugin marketplace add useSage/compass-marketplace

# Install the Compass plugin
/plugin install compass@sage
```

Keep it up to date:

```bash
/plugin marketplace update
```

## What's inside

The `compass` plugin bundles three skills:

| Skill | Invoke | What it does |
|-------|--------|--------------|
| `cli` | `/compass:cli` | Operate the platform with the `compass` CLI — run/follow executions, inspect workflows, executions, approvals, connections, projects. |
| `build-workflow` | `/compass:build-workflow` | Author a multi-node workflow definition against the Compass schema, then validate and deploy it. |
| `build-agent` | `/compass:build-agent` | Author an agent (instructions, model, tools) and deploy it. |

Skills also auto-trigger when your request matches — e.g. asking Claude to "build a
Compass workflow that…" loads `build-workflow` on its own.

## Prerequisites

- **Claude Code** with plugin support.
- The **`compass` CLI** installed and authenticated. Download the wheel from your
  Compass platform at **Settings → Developer → SDK & CLI** (`/settings/sdk`), then:
  ```bash
  uv tool install "./compass_core-<version>-py3-none-any.whl[cli]"
  compass auth login --url <your-compass-url>
  ```
  Or export `COMPASS_API_KEY` and `COMPASS_BASE_URL` instead of `auth login`
  (see the `cli` skill).

## Repository layout

```
compass-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # catalog: lists the plugins in this repo
└── plugins/
    └── compass/
        ├── .claude-plugin/
        │   └── plugin.json        # plugin metadata + version
        └── skills/
            ├── cli/
            ├── build-workflow/
            └── build-agent/
```

## Releasing updates

Updates are **pull-based**: users only receive a new version when they run
`/plugin marketplace update`, and only when the plugin's `version` changes.

1. Edit skills under `plugins/compass/skills/`.
2. Bump `version` in `plugins/compass/.claude-plugin/plugin.json` (semver).
3. Commit and push to `main`.

Because the skills wrap the separately-versioned `compass` CLI, they describe stable
command shapes and ground themselves against `compass --help` at runtime rather than
hardcoding an exhaustive flag list — so a CLI update doesn't silently break them.
