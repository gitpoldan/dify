# Fusion Query Engine — Dify Plugin

LlamaIndex-based tool plugin for Dify that performs **windowed COMPACT synthesis** over pre-fetched context (Qdrant points or plain text), with optional chat-history condensing.

## Features

- **Two modes**
  - `query_expansion` — enriches the user query using domain context (COMPACT over chunks)
  - `answer_query` — generates a detailed answer from context (COMPACT over chunks)
- **Pre-fetched context** — no Qdrant/embedding calls inside the plugin; nodes come from upstream workflow steps
- **Chat history** — `CondenseQuestionChatEngine` condenses multi-turn dialogue before synthesis
- **Configurable prompts** — four synthesis prompts + condense prompt with MegaDoc defaults
- **Two LLM selectors** — separate models for expansion and answer/condense steps

## Architecture

```
Workflow (Knowledge Retrieval / Code node)
        │
        ▼
  source_nodes  ──►  FlashContextRetriever
  or context_text              │
                               ▼
                    RetrieverQueryEngine (COMPACT)
                               │
                               ▼
                 CondenseQuestionChatEngine
                     (chat_history)
                               │
                               ▼
                         LLM text output
```

## Parameters (summary)

| Parameter | Description |
|-----------|-------------|
| `query` | User question |
| `mode` | `query_expansion` or `answer_query` |
| `model_expansion` | LLM for expansion mode |
| `model_answer` | LLM for answer mode and condense step |
| `source_nodes` | Array of Qdrant points (priority) |
| `context_text` | Plain-text fallback |
| `chat_history` | JSON chat turns |
| `context_window` | Default `112000` |
| `num_output` | Default `2500` (answer mode) |
| `expansion_num_output` | Default `350` (expansion mode) |
| `chunk_size_limit` | Default `15000` |
| `chunk_overlap_ratio` | Default `0.05` |

## Source nodes format

Priority input (`array[object]`). Each element is a Qdrant-style point:

```json
[
  {
    "id": "doc-chunk-1",
    "payload": {
      "text": "Document fragment text…",
      "metadata": {
        "file_name": "manual.pdf",
        "page": 12
      }
    },
    "score": 0.91
  }
]
```

If `source_nodes` is empty but `context_text` is set, the plugin wraps the text as a single node (`score: 1.0`).

## Chat history wiring in Chatflow

Tool plugins do not expose native `history-messages` (that is agent-strategy only). Pass history explicitly:

1. Create a **conversation variable** `chat_history` (type: array/object or string).
2. After each turn, append `{"role":"user","content":…}` and `{"role":"assistant","content":…}` via **Variable Assigner** or a **Code** node.
3. Serialize to JSON string and map it to the tool parameter `chat_history`.

Example JSON:

```json
[
  {"role": "user", "content": "Что такое процесс X?"},
  {"role": "assistant", "content": "Процесс X — это…"}
]
```

In the tool node, set:

- **query** ← `{{sys.query}}` or LLM output
- **chat_history** ← `{{#conversation.chat_history#}}` (your variable)
- **source_nodes** ← array variable from Knowledge Retrieval or a Code node

## Example workflow

1. **Knowledge Retrieval** — fetch chunks from Dify dataset
2. **Code** (optional) — normalize hits to Qdrant point objects → `source_nodes`
3. **Tool: Fusion Query Engine**
   - `mode`: `answer_query`
   - `query`: user question
   - `chat_history`: conversation variable
4. **Answer** — show tool output

For a two-step RAG pipeline:

1. Run tool with `mode=query_expansion` → enriched query
2. Run Knowledge Retrieval with enriched query
3. Run tool with `mode=answer_query` → final answer

## Local development

```bash
pip install -r requirements.txt
cp .env.example .env
# fill REMOTE_INSTALL_KEY from https://debug.dify.ai
python -m main
```

Install the plugin in Dify via remote debug or package upload.

## Packaging (.difypkg)

Dify expects **`manifest.yaml` in the root of the zip archive**, not inside a subfolder.

Wrong (causes `open manifest.yaml: file does not exist`):

```
llamaindex_FusionQueryEngine/
  manifest.yaml
  main.py
  ...
```

Correct:

```
manifest.yaml
main.py
provider/
tools/
_assets/icon.svg
...
```

From PowerShell (run inside the plugin project directory):

```powershell
Compress-Archive -Path * -DestinationPath ..\fusion_query_engine.difypkg -Force
```

Or with the Dify CLI from the plugin root:

```bash
dify plugin package ./
```

Do not zip the parent folder itself — only its contents. Exclude `__pycache__`, `.venv`, and `.env` (see `.difyignore`).

## Dependencies

- `dify_plugin` **≥ 0.9.1** (Python 3.12+, Dify ≥ 1.14.2)
- `llama-index-core` — COMPACT synthesis, chat engine, retriever abstractions

## SDK / Dify compatibility

| Component | Requirement |
|-----------|-------------|
| Python | 3.12+ (plugin runner) |
| Dify | ≥ 1.14.0 (`meta.minimum_dify_version`; prod k8s: `dify-api`/`dify-web` **1.14.0**) |
| dify_plugin | ≥ 0.9.1, &lt; 1.0.0 |

SDK 0.9.x mainly adds **LLM polling** for model-provider plugins (not used here). This plugin uses reverse LLM invoke via `session.model.llm`, which remains the same API.

## Workflow output variables

The tool declares an `output_schema` and returns named variables for downstream nodes:

| Variable | Description |
|----------|-------------|
| `response` | Final LLM text (answer or enriched query) |
| `mode` | `query_expansion` or `answer_query` |
| `enriched_query` | Set only in expansion mode; use as input to Knowledge Retrieval in a two-step RAG pipeline |

Example: `{{#fusion_query_engine.enriched_query#}}` → Knowledge Retrieval → second tool call with `mode=answer_query`.

## Privacy

This plugin does not persist user data. Context and queries are sent to the LLM providers configured in your Dify workspace. See [PRIVACY.md](PRIVACY.md).
