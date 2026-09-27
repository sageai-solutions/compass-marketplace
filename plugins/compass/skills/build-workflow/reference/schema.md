# WorkflowDefinition schema reference

The shape of a Compass workflow definition (YAML or JSON). Two live oracles
outrank this file when they disagree: `compass nodes list --json` (the full
palette: components + agent + integrations; `compass components get <type>
--json` for one component's depth), and `compass workflows validate <id>` as
the final word — map its error codes with the table at the bottom.

## Top-level fields

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `name` | str | **required** | Missing → parser error. |
| `type` | str | `"workflow"` | `"workflow"` \| `"agent"`. |
| `description` | str | `""` | |
| `input_schema` | dict | `{}` | Usually derived from the Input node; leave empty. |
| `version` | int | `1` | |
| `nodes` | list | `[]` | The graph. Each node needs at least `id` and `type`. |

## Node base fields (`nodes[]`)

| Field | Type | Notes |
|-------|------|-------|
| `id` | str | **required**, unique. |
| `type` | str | **required**. One of the 4 node types below. |
| `name` | str | Display label. |
| `inputs` | dict[str, Binding] | Param name → binding. An unbound param is simply **omitted**. |
| `config` | dict | Component/agent config (see below). |
| `output_schema` | dict? | Optional override. |
| `position` / `meta` | — | Canvas-only; ignored by execution, safe to omit. |
| `depends_on` | list | **DERIVED from `ref`/`template`/`list`/`object` bindings. NEVER author it.** |

## The 4 node types

| `type` | Extra fields |
|--------|--------------|
| `component` | `component_type: str`, `config: dict`. The building blocks (input/output/prompt/…). |
| `agent` | `agent_id` (reference a prebuilt agent — then inline fields are ignored) **or** inline: `model`, `system_prompt`, `tools`, `settings`, `reasoning`, `conversation_history`, `response_format`, `batch_over`, `batch_concurrency`, `tools_require_approval`. See **Agent node** below. |
| `integration` | `integration_type: str`, `connection_id: str?`, `action: str`. Calls an external OpenAPI connector. |
| `workflow` | `workflow_id: str?`. Runs another workflow as a sub-step. |

## The component types (`component_type`)

Snapshot — `compass components list --json` gives the live catalog, and
`compass components get <type> --json` the full spec per component.

Components carry **capabilities**, exactly like integrations: each is a named action with
its own input and output schema. A component node names the one it runs as `action`, which
may be left out when the component has only one. `input` and `output` are the workflow's
entry and exit and have none. Attached to an agent as a tool, a component exposes one tool
per capability, narrowed with `enabled_capabilities` / `approval_capabilities` as an
integration tool is.

| `component_type` | Capabilities | Purpose | Key `config` |
|------------------|--------------|---------|--------------|
| `input` | — | Entry point (exactly one). | `type: chat` (default) \| `object`. `object` needs `config.schema` (see below). |
| `output` | — | Exit point (exactly one). | `mode: single` (default, value unwrapped) \| `structured` (object of all fields); `fields: [{name, type: {type, json_schema?}}]`. |
| `prompt` | `render` | Renders a template string. | `template` (**required**); bare `{{name}}` tokens auto-declare mapper-bound inputs. |
| `document_splitter` | `split` | Split a document by pages (tool-capable). | `pages_per_chunk`=1. Input requires `document`. |
| `document_parser` | `parse` | Parse/OCR a document (tool-capable). | `always_ocr`=false, `allow_unreadable_pages`=false, `batch_concurrency`=5. Input requires `document`. |
| `embeddings` | `embed` | Embed text. A bare-string input is chunked first; output is always a list. | `text_field`="text", `chunk_size`=1000, `chunk_overlap`=200, `split_on_headings`=true, `batch_size`=100, `batch_concurrency`=5. Input requires `input`, `model`. |

### Input node `config.type: object`
```yaml
config:
  type: object
  schema:
    topic:   { type: string,  description: "...", required: true }
    count:   { type: integer, required: false }
    doc:     { type: file }        # file -> attachment descriptor
```
Field `type` ∈ `string | integer | number | boolean | object | array | file`.
Input node has **no** `inputs` and **no** `depends_on`.

### Output node
Each entry in `config.fields` becomes a bindable param in `inputs`. `mode: single`
returns the one field's value unwrapped; `mode: structured` returns an object of
all fields. Output must depend on ≥1 node.

## The 6 binding kinds (values of `inputs[param]`)

| `kind` | Fields | Meaning |
|--------|--------|---------|
| `literal` | `value: Any` | A constant. |
| `ref` | `source: str`, `path: str` | Pull from an upstream node. `path` is a JSONPath subset: `$`, dot-access, integer index — **no wildcards/filters**. |
| `template` | `value: str` | Free text with `{{node.path}}` tokens (string params only). |
| `list` | `items: [Binding]` | A list of bindings. |
| `object` | `entries: {str: Binding}` | An object of bindings. |
| `config` | `key: str`, `optional: bool` | Value supplied per-run in the run `config` dict (system workflows only). |

## Agent node (inline)

```yaml
- id: agent
  type: agent
  model:                       # REQUIRED (see model rule below)
    connection_id: <uuid>
    model: ""                  # "" = the connection's configured model
  system_prompt: "..."
  settings: { temperature: 0.7, top_p: 1.0, max_tokens: 1024 }   # optional, verbatim ModelSettings
  reasoning: null              # optional ReasoningEffort
  conversation_history: false
  response_format: null        # optional JSON Schema constraining the agent's output
  batch_over: ""               # "" off | "$" input is the list | "<field>" that field is the list
  batch_concurrency: 5
  tools_require_approval: false
  tools: []                    # list of ToolSpec (agent:/workflow:/integration: sources)
  inputs:
    input: { kind: ref, source: <upstream-id>, path: "$" }
```

## THE MODEL RULE (most common deploy failure)

An inline agent node **must** have a `model`, and that model must resolve to a
provider connection that has a model configured. Two gates:

1. **Build gate** — `model: null` on an inline agent → error `MISSING_CONNECTION`
   ("Agent node needs a model").
2. **Deploy gate** — the referenced connection must exist, be active, and have a
   model configured, or → error `MISSING_MODEL`.

So before authoring, get a real provider connection id:
```
compass providers list --json      # or: compass connections list --json
```
Put it in `model.connection_id`. Leave `model.model` empty to use the connection's
own configured model, or set it to pin a specific model. A node that uses
`agent_id:` instead carries **no** model of its own — the referenced agent supplies it.

## Validation error codes (from `compass workflows validate`)

**Structural (build):** `MISSING_INPUT`, `DUPLICATE_INPUT`, `MISSING_OUTPUT`,
`DUPLICATE_OUTPUT`, `INPUT_HAS_DEPENDENCIES`, `OUTPUT_NO_DEPENDENCIES`,
`UNKNOWN_DEPENDENCY`, `CYCLE_DETECTED`, `UNKNOWN_COMPONENT_TYPE`,
`MISSING_COMPONENT_CONFIG`, `INVALID_COMPONENT_CONFIG`, `PROMPT_PARAM_HAS_PATH` (warn).

**Deploy-time (ready):** `MISSING_CONNECTION`, `MISSING_MODEL`, `CONNECTION_NOT_FOUND`,
`CONNECTION_INACTIVE`, `CONNECTION_CAPABILITY`, `DOES_NOT_REACH_OUTPUT` (warn),
`CONNECTED_NOT_ASSIGNED` (warn), `MISSING_REQUIRED_PARAMETER`, `TYPE_MISMATCH`,
`INVALID_BATCH_AXIS`, `INVALID_BATCH_CONFIG`.

**Agent entities (`type: agent`) only:** `AGENT_SHAPE` (exactly one inline agent),
`NESTED_AGENT` (an agent may not reference another agent), `MISSING_CONNECTION`
(needs a model), plus Output must consume the agent.
