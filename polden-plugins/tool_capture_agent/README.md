# Tool-Capture Agent вЂ” Dify Plugin

Author: **polden** · Version: **0.3.3** · Requires Dify **≥ 1.16.1**

A customized **Agent Strategy** plugin for Dify, built on top of Dify's official
Function-Calling agent. It behaves like the standard **Agent** node (same model, tools,
instruction, query, iterations, chat history), and adds:

> **1. Capture** — every tool call is returned as addressable workflow variables
> (`tool_calls`, `tool_outputs`, `mapped_outputs`, …).
>
> **2. I/O wiring** — optional JSON `io_wiring` links outputs→inputs through runtime
> indexes so large payloads do **not** have to pass through the LLM context (stubs +
> auto-injected tool args).
>
> **3. Incoming files** — wire strategy parameter **`files`** → `{{#sys.files#}}` so
> Chatflow attachments become multimodal user-prompt content (stock FC only sends text).
>
> **4. Skills & Tools routing** — optional YAML catalogs; a cheap pre-call picks the
> relevant entries, assembles the system prompt from their details and narrows the tool
> set handed to the model.

## Why

The standard Dify Agent node exposes essentially two workflow outputs: `text` (the final
answer) and a `json` metadata blob. The intermediate tool results are only visible in the
run logs and cannot be referenced by downstream nodes. This plugin lifts those tool
outputs into first-class, typed variables.

## Plugin type

- **Type:** `agent-strategy` (`plugins.agent_strategies`)
- **Strategy:** `function_calling` вЂ” Function Calling with tool-output capture
- **Backwards invocation:** tools + LLM (same permissions as the official agent)

## Output variables

| Variable | Type | Description |
|----------|------|-------------|
| `text` | string | Streamed final answer (standard agent output). |
| `final_answer` | string | Final assistant answer as an addressable variable. |
| `tool_calls` | array | Ordered list of **every** tool call (see record shape below). |
| `tool_outputs` | object | `{ tool_name: [response, response, ...] }` вЂ” all responses grouped by tool. |
| `last_tool_outputs` | object | `{ tool_name: last_response }` вЂ” convenient for single-call tools. |
| `tool_calls_count` | number | Total number of executed tool calls. |
| `mapped_outputs` | object | User-defined variables resolved from `output_mappings`. |
| `inputs_index` | object | Addressable agent + tool input slots for this run. |
| `outputs_index` | object | Addressable outputs (values filled as tools run). |
| `io_wiring_applied` | object | Parsed wiring config, validation errors, per-call resolve log. |
| `captured_files` | array | Metadata (`url`, `filename`, `mime_type`, ...) of every file produced by tools. (Not named `files` вЂ” that name is reserved by Dify.) |
| `tool_files` | object | `{ tool_name: [file_meta, ...] }` вЂ” file metadata grouped by tool. |
| `files_count` | number | Files **produced by tools** in this run (not incoming `sys.files`). |
| `execution_metadata` | object | Token usage + pricing for the whole run (router pre-call included). |
| `skills_tools_routing` | object | Router decision: selected ids, pinned tools, resulting tool whitelist, parse/fallback notes. |

## Files & download cards

### Incoming attachments (`sys.files`)

Wire Chatflow attachments directly into the strategy:

```
Agent → Files ← {{#sys.files#}}
```

Dify ≥ 1.16 converts `File` → plugin dict before invoke. On the first LLM turn the
strategy injects images/documents as multimodal prompt content (vision model required
for images). Optional **`image_detail`**: `high` (default) or `low`.

The first user message also starts with a short plain-text list of the attachments:

```
[Attached files]
1. Планирование авг 2026.xlsx (xlsx, document, 9.7 KB)
Images are shown to you directly, other formats may not be — read them with a
file-processing tool. Attachments are passed to tools automatically, so never ask
the user to attach the file again.
```

This is not cosmetic. The `langgenius/openai_api_compatible` provider builds the request
from text and image parts only, so a document part is dropped without a trace and the
model answers that no file was given — it never calls the file tool. The text list always
reaches the model, whatever the provider does with the multimodal parts.

**External LiteLLM / vLLM:** in-cluster cards often carry `http://api:5001/files/...`.
Set **Public files base URL** to your public origin (e.g. `https://dify-test.protek.ru`)
so only the multimodal prompt URL is rewritten; **Internal files base URL** defaults to
`http://api:5001`. Leave public empty if the model runs inside the cluster. Tool /
`io_wiring` cards keep the original internal URL.

Do not bind `sys.files` to a **tool** FORM file parameter inside the Agent — nested
`File` in `tools[].runtime_parameters` is still not converted by Dify and can crash
plugin JSON. Use `io_wiring` (`agent.files` → `tool.<name>.…`) or pass URLs to tools.

### Tool-generated files (outputs)

Tool-generated files (documents, images, archives, ...) are **forwarded as file
messages**, so they populate the Agent node's native **`files`** output (type
`Array[File]`). To show a downloadable file card in chat, reference that output in an
Answer node:

```
{{#agent.files#}}
```

> The upstream official Function-Calling strategy drops non-image tool files (they never
> reach the node output). This plugin forwards them, which is what makes the file card
> render.

In addition, the plugin exposes **file metadata** (URLs, names, mime types) as plain data
via the `captured_files` / `tool_files` output variables, handy for building custom links
or routing logic. Note these are metadata objects, not `File`-typed values вЂ” the native
`files` node output is the one that renders cards. (The metadata variable is called
`captured_files` rather than `files` because `files` is a reserved output name in Dify.)

Each element of `tool_calls` looks like:

```json
{
  "index": 1,
  "iteration": 1,
  "tool_call_id": "call_abc",
  "tool_name": "get_weather",
  "provider": "weather_tools",
  "arguments": { "city": "Berlin" },
  "response": "tool response: {\"temp\": 21}.",
  "response_json": [ { "temp": 21 } ],
  "is_error": false,
  "started_at": 12345.6,
  "elapsed_time": 0.42
}
```

## Skills & Tools routing

Large agents die from prompt bloat: every skill and every tool manual has to sit in the
system prompt at once, and the model loses focus. Fill **Skills catalog (YAML)** and/or
**Tools catalog (YAML)** and the strategy runs one cheap LLM pre-call that selects a
working set for the current request. Both fields empty → behaviour is identical to 0.2.x.

```yaml
# Skills catalog (YAML)
- id: refund
  name: Refund policy
  summary: Handle refunds and returns          # seen by the router
  detail: |                                     # injected into the system prompt if selected
    Ask for the order id, check the 14-day window, then call the ERP tool.
  requires: [orders]
  requires_tools: [erp]   # tool catalog ids pulled in with this skill
- id: orders
  summary: Look up orders in the ERP
  detail: Orders live in the ERP; ids look like ORD-0000.
- id: tone
  summary: Tone of voice
  detail: Always answer politely, in the user's language.
  always: true        # never routed away
```

```yaml
# Tools catalog (YAML)
- id: search
  tool_name: tavily_search      # real tool name; defaults to id
  summary: Web search for fresh facts
  detail: Pass a short query, cite the urls you used.
- id: erp
  summary: ERP lookup
  detail: Needs an order id.
- id: fusion
  tool_name: FusionAllContextToResponse
  summary: Compose the final answer from collected context
  detail: Call once, last.
```

How it runs:

1. **Instruction** stays the mission statement — it is sent to the router and prefixed to
   the assembled prompt.
2. The router sees only `id` + `name` + `summary` of every entry (`always` entries are
   hidden from it — they are included unconditionally), the last 6 history messages
   (text only), the names/kinds of files attached to this turn, and the current query.
3. Selection is closed over skill `requires`, then skill `requires_tools` are merged into
   the tools selection (which also closes over tool `requires`). `always` entries are
   added, then trimmed to **Max selected per catalog** (default 8).
4. The system prompt becomes `instruction` + `detail` of the selected skills + `detail` of
   the selected tools. Only the selected tools' schemas reach the model.

Guarantees worth knowing:

- Tools attached to the node but **absent from the catalog** are always available — the
  catalog is an instruction layer, not an allow-list.
- Tools **already called** earlier in the conversation are pinned (history `tool_call`
  messages must stay valid). `io_wiring` does **not** pin tools — wiring applies at
  invoke time; declare `requires_tools` on skills when a tool must ride with a skill.
- If the pre-call fails or returns non-JSON, the strategy **falls back** to the full
  catalog and all tools, and records why in `skills_tools_routing.errors`.
- **Router model** (optional) points the pre-call at a cheap/fast model; otherwise the
  main model is reused with `temperature=0` and **Router max tokens** (default 512).
  Its usage is folded into `execution_metadata`.

Inspect a run via `{{#agent.skills_tools_routing#}}` or the `ROUTING skills & tools` log
entry in the node trace.

## I/O wiring: `io_wiring`

Use this when tools return large JSON/text and chaining them through the model blows the
context window (or prevents tool calls). Dify Agent Strategy UI has **no** click-to-add
from/to form вЂ” configure wiring as JSON:

```json
{
  "wires": [
    { "from": "agent.query", "to": ["tool.search.query"] },
    { "from": "tool.search", "to": ["tool.summarize.text"] }
  ],
  "context_mode": "stubs",
  "stub_max_chars": 240,
  "tool_context": {
    "FusionAllContextToResponse": ["fusion_answer", "sources_metadata"]
  }
}
```

- Wire targets are **managed**: filled by the strategy, stripped from the LLM tool schema.
- Full results live in `outputs_index` / capture variables; the model sees a short stub
  when the response is longer than `stub_max_chars`.
- **`tool_context`**: map a tool name to a JSON field (string shorthand), a
  **list of fields**, or `{"field"|"fields": "...", "max_chars":12000}`.
  Projected value(s) are injected into the model's `ToolPromptMessage` as the
  tool's response — so the agent can treat e.g. `fusion_answer` (and optionally
  `sources_metadata`) as the answer of `FusionAllContextToResponse` even in stub
  mode. Multiple fields are joined with a blank line, without field-name
  headings; missing/empty fields are skipped.
- Inspect `io_wiring_applied` after a run for validation / resolve errors and available
  index keys.
- **Use straight double quotes** (`"`). The parser tolerates smart/typographic quotes
  (`вЂњ вЂќ вЂ вЂ™ В« В»`) and Python-style single quotes, but records a note in
  `io_wiring_applied.errors`. Invalid JSON errors now echo the first characters actually
  received, so you can spot editor quote-substitution.
- **Fan-out** (one `from` в†’ many `to`): same value copied to every target.
- **Fan-in** (many `from` → one `to`): strategy field **Fan-in mode**
  (`last_nonempty` | `first_nonempty` | `merge`). JSON key `fan_in_mode` overrides the
  field. `merge` does list concat / string concat / dict shallow update (JSON object
  strings are parsed when possible). Notes land in `io_wiring_applied.errors`.
- **Tool context mode** (`stubs` | `full`): what the model sees after each tool
  call. JSON key `context_mode` overrides the field.
- **`optional: true` on a wire**: if the source is empty, skip that target (no hard fail);
  the model may supply alternatives like `context_text`. Required unresolved wires
  **soft-fail** (stub to the model, tool not invoked). A DAG prerequisite hint is
  appended to the system instruction automatically.

## Extending the output set: `output_mappings`

The `output_mappings` parameter lets you assign specific tool outputs to your own named
variables. Provide a JSON object where each **key** is the variable name and each **value**
is either a tool name (shorthand) or a rule object:

```json
{
  "weather": "get_weather",
  "answer": { "tool": "FusionAllContextToResponse", "field": "fusion_answer" },
  "all_search_hits": { "tool": "tavily_search", "occurrence": "all", "field": "json" },
  "first_price": { "tool": "get_price", "occurrence": "first", "field": "response" }
}
```

Rule object fields:

| Field | Values | Default | Meaning |
|-------|--------|---------|---------|
| `tool` | tool name | вЂ” | Which tool's output to read. |
| `occurrence` | `last` \| `first` \| `all` \| `<int>` | `last` | Which call(s) to use (`all` returns a list; `<int>` is a 0-based index). |
| `field` | `auto` \| `response` \| `json` \| `arguments` \| `<dot.path>` | `auto` | What to extract. `auto` prefers structured JSON; any other non-reserved string is a dotted path into the JSON (e.g. `fusion_answer`, `data.items.0.id`). |

Resolved values are always available under `mapped_outputs` (e.g.
`{{#agent.mapped_outputs.weather#}}`). On Dify versions that expose dynamic agent output
variables, each key is **also** emitted as its own variable (e.g. `{{#agent.weather#}}`).

## Usage in a workflow

1. Add an **Agent** node and select strategy **Tool-Capture Agent в†’ Function Calling
   (Tool-Capture)**.
2. Configure it like the normal agent: Model, Tools, Instruction, Query, Maximum
   Iterations.
3. (Optional) Fill **Skills catalog (YAML)** / **Tools catalog (YAML)** to route a compact
   system prompt and a focused tool set per request.
4. (Optional) Fill **I/O wiring (JSON)** to pass tool data via indexes (stubs in context).
5. (Optional) Fill **Output mappings (JSON)** to name specific tool outputs for the
   workflow.
6. In a downstream **Answer** / **Template** / **LLM** node, compose a flexible response
   from the new variables, e.g.:

```
Weather: {{#agent.mapped_outputs.weather#}}
Sources: {{#agent.tool_outputs.tavily_search#}}
Total tool calls: {{#agent.tool_calls_count#}}
```

## Local development

```bash
pip install -r requirements.txt
cp .env.example .env       # fill REMOTE_INSTALL_KEY from your Dify в†’ Plugins в†’ Debug
python -m main
```

## Packaging (.difypkg)

Dify expects `manifest.yaml` at the **root** of the archive (not inside a subfolder).
Do **not** use PowerShell `Compress-Archive` - it can produce backslash paths / wrong
`create_system` metadata that Dify rejects. Use the packager in this folder instead:

```bash
# run inside this plugin directory
python package.py
```

This writes `../polden-tool_capture_agent_0.3.3.difypkg` with forward-slash entries and
Unix `create_system=3`.

or with the Dify CLI:

```bash
dify plugin package ./
```



## Compatibility

| Component | Requirement |
|-----------|-------------|
| Python | 3.12+ (plugin runner) |
| Dify | в‰Ґ 1.7.0 (agent-strategy plugins); tested against 1.14.x deployments |
| dify_plugin | в‰Ґ 0.9.1, < 1.0.0 |

## Privacy

The plugin persists nothing. Captured tool outputs live only for the duration of one
Agent node execution. See [PRIVACY.md](PRIVACY.md).
