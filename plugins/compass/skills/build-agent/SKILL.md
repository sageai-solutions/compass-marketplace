---
name: build-agent
description: >-
  Author a Compass agent — instructions/system prompt, a model via a provider
  connection, and optional tools — then deploy it with the compass CLI. Use when
  the user wants to build, create, or configure an agent, an LLM assistant, a
  chatbot, or a single-step AI task on Compass, or to give an agent tools. For a
  multi-node pipeline (splitters, parsers, multiple steps), use build-workflow
  instead.
allowed-tools: "Bash(compass:*) Write Read"
---

# Building a Compass agent

An agent is a `type: agent` workflow: one inline agent node between an input and an
output. The `compass agents create` command compiles flat flags into that graph for
you — so the flags path is the clean, recommended way to build one.

If `compass --version` fails, stop — the CLI isn't installed (see the `cli` skill).

## Step 1 — Get a provider connection (do this first)

The agent's model comes from a **provider connection**, not a bare model name. Find one:
```
compass providers list --json      # or: compass connections list --json
```
Note an active connection's `id`. If there are none, tell the user to create a
provider connection first — you can't invent a model. Without a working model the
agent won't deploy (`MISSING_CONNECTION` / `MISSING_MODEL`).

## Step 2 — Create the agent (flags path, recommended)

Non-interactively, `create` requires at minimum `--name`, `--provider`, and
`--instructions`:
```
compass agents create \
  --name "Support Triage" \
  --provider <provider-connection-id> \
  --instructions "You triage support tickets. Classify severity and suggest an owner." \
  --json
```

| Flag | Meaning |
|------|---------|
| `--name` | Agent name. |
| `--description` | One-line summary. |
| `--instructions` | **The system prompt.** |
| `--provider` | **Provider connection ID** — supplies the model. |
| `--tool` | Repeatable. Format `agent:<ID>` \| `workflow:<ID>` \| `integration:<CONN_ID>`. |
| `--conversation-history` / `--no-conversation-history` | Multi-turn memory (default off). |
| `-f`, `--file` | A full agent **definition YAML** instead of flags (see Step 2b). |

Capture the returned agent `id`.

> Omitting required flags with an interactive terminal launches a wizard; in a
> non-interactive run you must pass the flags. Confirm exact flags with
> `compass agents create --help`.

## Step 2b — Full control (definition file)

For custom `settings` (temperature, max_tokens), a `response_format` (structured
output), tools that require approval, or batching, author the agent **definition YAML**
and pass it with `-f`:
```
compass agents create -f agent.yaml --json
```
Use `type: agent` and the agent-node schema from the build-workflow reference:
[../build-workflow/reference/schema.md](../build-workflow/reference/schema.md).
Agent shape is strict: exactly one inline agent node (no `agent_id`), it must have a
`model`, it may not reference another agent, and the Output must consume it.

## Step 3 — Test, then deploy

Run the draft to try it (note: on `execute`, `-f` means **--follow**, not a file):
```
compass agents execute <agent-id> -m "Ticket: login page 500s for all users" -f
```
Then deploy (deploy works on an agent's id):
```
compass workflows deploy <agent-id> --json
```

## Rules

- Ground uncertain flags with `compass agents --help` / `compass agents create --help`.
  Watch the `-f` collision: `create`/`update` → file; `execute` → follow.
- Show the user the instructions, model, and tools, and confirm before creating/deploying.
- Deploy and delete change real state — confirm intent first.
- Never print or embed the API key.
