---
name: build-workflow
description: >-
  Author a multi-node Compass workflow definition (input → components/agents →
  output) against the WorkflowDefinition schema, then validate and deploy it with
  the compass CLI. Use when the user wants to build, create, author, or design a
  Compass workflow, wire nodes together, define a workflow's inputs and outputs,
  turn a described process or pipeline into a deployable Compass workflow, or fix
  a workflow that fails validation. For a single-agent assistant, use build-agent
  instead.
allowed-tools: "Bash(compass:*) Bash(jq:*) Write Read"
---

# Building a Compass workflow

A workflow is a directed graph of nodes serialized as a YAML/JSON
`WorkflowDefinition`. You author the definition, create/update the workflow entity,
then validate and deploy it — all through the `compass` CLI.

**The full schema is in [reference/schema.md](reference/schema.md). Read it before
authoring.** A copy-ready starting point is
[reference/example-workflow.yaml](reference/example-workflow.yaml).

If `compass --version` fails, stop — the CLI isn't installed (see the `cli` skill).

## Step 1 — Get a model connection (do this first)

Every agent node needs a provider connection that has a model configured, or deploy
fails with `MISSING_CONNECTION` / `MISSING_MODEL`. Find one up front:
```
compass providers list -o json      # or: compass connections list -o json
```
Pick an active connection's `id`. If there are none, tell the user to create a
provider connection first (in the platform UI, or `compass providers create -f …`);
you can't invent a model.

## Step 2 — Author the definition

Write `definition.yaml` following [reference/schema.md](reference/schema.md). The
non-negotiables:

- Exactly **one** `input` component and **one** `output` component.
- **Never** author `depends_on` — it's derived from your `ref`/`template`/`list`/
  `object` bindings.
- Wire nodes with `ref` bindings: `{ kind: ref, source: <upstream-id>, path: "$" }`.
- Put the connection id from Step 1 into each agent node's `model.connection_id`.

Sketch the graph for the user in plain terms before writing YAML, then write it.

## Step 3 — Create (or update) the workflow entity

**Editing an existing workflow** — `update -f` takes the raw definition text:
```
compass workflows update <workflow-id> -f definition.yaml -o json
```

**Creating a new workflow** — `create -f` expects a full **Workflow envelope** (JSON
with the definition embedded as a string), not the bare definition. Wrap it with jq:
```
jq -Rs '{clerk_org_id:"placeholder",project_id:"00000000-0000-0000-0000-000000000000",name:"Summarizer",description:"",type:"workflow",definition:.}' \
  definition.yaml > envelope.json
compass workflows create -f envelope.json -o json
```
`clerk_org_id`/`project_id` are placeholders — the server overrides them from your
auth scope; they only need to be present to pass client-side validation. Capture the
returned `id`. (If `create` rejects the envelope, run `compass workflows create --help`
and adjust — or GET an existing workflow's JSON, swap `name` + `definition`, and re-POST.)

## Step 4 — Validate and fix

```
compass workflows validate <workflow-id> -o json
```
Map any error codes using the table in [reference/schema.md](reference/schema.md)
(e.g. `MISSING_MODEL` → the connection has no model; `DUPLICATE_INPUT` → more than one
input component). Edit `definition.yaml`, re-run `update -f`, re-validate. Loop until
clean.

## Step 5 — Deploy and run

```
compass workflows deploy <workflow-id> -o json
compass execute -d <deployment-id> -i topic="Q3 report" -f    # run the deployment
```
To test before deploying, execute the draft directly:
```
compass execute -w <workflow-id> -i topic="Q3 report" -f
```

## Rules

- Ground uncertain flags with `compass workflows --help` / `<cmd> --help` before running.
- Show the user the definition and confirm before creating/deploying anything.
- Deploy and delete change real state — confirm intent first.
- Never print or embed the API key.
