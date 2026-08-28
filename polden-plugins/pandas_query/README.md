# Pandas Query Engine — Dify Plugin

Process **CSV / XLSX** with the real [LlamaIndex Pandas Query Engine](https://developers.llamaindex.ai/python/examples/query_engine/pandas_query_engine/) (vendored from `llama-index-experimental` 0.5.5) + `llama-index-core`. Extra modes: quality / transform / chart.

Author: **polden** · version **0.0.9**

> **Important:** the full spreadsheet is **not** sent to the LLM. Only `df.head(N)` goes into the prompt; generated pandas code runs locally on the whole DataFrame.

> **Multi-turn ChatFlow:** set `session_key` to `{{#sys.conversation_id#}}`. The file uploaded on the first turn is cached in the plugin's **persistent storage** and reused on later turns — no File-typed conversation variable needed, no re-upload.

> **Agent tip:** input is `files` (Array[File]), not singular `file`. On follow-up turns Dify may pass several files; the plugin picks the CSV/XLSX automatically.

## Features

| Mode | What it does | Typical output |
|------|----------------|----------------|
| `auto` | Classifies the task with the LLM, then routes | depends |
| `analyze` | LlamaIndex `PandasQueryEngine` → pandas expression → answer | text |
| `quality` | Deterministic profile (nulls, dtypes, duplicates, …) + optional LLM narrative | text + `quality_report` |
| `transform` | Generates pandas code, runs it, exports the resulting DataFrame | `.xlsx` / `.csv` + text |
| `chart` | Generates matplotlib code, returns a PNG | `.png` (+ optional table) |

Also includes:

- **Dify LLM reverse-invoke** adapter (same pattern as `llamaindex_FusionQueryEngine`)
- **`delivery_mode`**: `file` (blob, Workflow) or `link` (upload + URL, Agent-safe)
- **`file` + `file_url`**: Workflow File object or Agent-friendly URL fallback
- Safety filter on generated code (blocked imports / dunders / `open` / network libs)

## Parameters (summary)

| Parameter | Description |
|-----------|-------------|
| `task` | Natural-language instruction |
| `files` | CSV / XLSX as Array[File] (Agent multi-turn safe) |
| `file_url` | URL fallback (Agent) |
| `session_key` | `{{#sys.conversation_id#}}` → cache file across ChatFlow turns |
| `model` | `model-selector` LLM |
| `mode` | `auto` / `analyze` / `quality` / `transform` / `chart` |
| `output_format` | `auto` / `xlsx` / `csv` for transform results |
| `delivery_mode` | `file` or `link` |
| `context_window` | default `112000` |
| `num_output` | default `2500` |
| `synthesize_response` | turn raw pandas output into prose |
| `sheet_name` / `csv_sep` | Excel sheet / CSV delimiter |

## Example tasks

- «Какой город с максимальной популяцией?» → `analyze`
- «Оцени качество данных и пропуски» → `quality`
- «Оставь только строки с amount > 1000 и сохрани» → `transform`
- «Построй bar chart продаж по месяцам» → `chart`

## Workflow tips

1. Start node → File upload (`document` / custom `.csv`/`.xlsx`)
2. Tool **Pandas Query Engine**
   - `files` ← uploaded file(s) / `{{#sys.files#}}`
   - `task` ← user query or LLM output
   - `mode` ← `auto` (or pin a mode)
   - `delivery_mode` ← `file`
3. Answer / End — show `response`; files appear in the tool node `files` output

For Agents: set `delivery_mode=link` and prefer `file_url` if File params are awkward.

## Multi-turn ChatFlow (upload once, ask many times)

Dify does not require a File-typed conversation variable for this — the plugin
caches the file itself in its persistent KV storage:

1. Enable file upload (Features) or a Start-node File variable.
2. Tool node mapping:
   - `files` ← `{{#sys.files#}}`
   - `session_key` ← `{{#sys.conversation_id#}}`  ← **the important part**
   - `task` ← `{{#sys.query#}}`
3. First message: user uploads the spreadsheet + asks → plugin caches it under
   the conversation id.
4. Next messages: user just asks; `sys.files` may be empty, but the plugin
   restores the cached file from storage by `session_key`.

Cache is scoped to the plugin's storage (quota 50 MB) and keyed by conversation.
Uploading a new file on a later turn overwrites the cache for that conversation.

## Packaging (.difypkg)

`manifest.yaml` must be at the **root** of the archive (not inside a subfolder).

Preferred (Unix-style zip paths):

```powershell
python package.py
```

Or from PowerShell inside the plugin directory:

```powershell
Compress-Archive -Path * -DestinationPath ..\polden-pandas_query_0.0.9.difypkg -Force
```

Or: `dify plugin package ./`

## Dependencies

- `dify_plugin` ≥ 0.9.1
- `llama-index-core` (same family as Fusion Query Engine) — **not** `llama-index-experimental`
- `pandas`, `openpyxl`, `matplotlib`, `numpy`, `httpx`, `tabulate`

> **Why no `llama-index-experimental`?** That package depends on `llama-index-finetuning` → **`torch` (~2GB)**, which times out on corporate Nexus. We vendor only the Pandas Query Engine sources (`engine/li_pandas/`, upstream 0.5.5) so behavior matches LlamaIndex without pulling torch. Do **not** “wait for torch” — it is unused by this engine.

## Compatibility

| Component | Requirement |
|-----------|-------------|
| Python | 3.12+ |
| Dify | ≥ 1.14.0 |
| dify_plugin | ≥ 0.9.1, &lt; 1.0.0 |

## Privacy

See [PRIVACY.md](PRIVACY.md). Generated code runs in a restricted sandbox; still treat this as non-production-hardened code execution.
