# Markdown to DOCX Pro — Dify Plugin

Convert Markdown content into a Microsoft Word **`.docx`** file directly inside Dify.

This plugin is a rework of the popular `stvlynn/doc` converter with three practical
additions and a more robust file return:

- **Narrow, configurable page margins** (`margin_cm`, default `1.5` cm on all sides)
- **Selectable built-in Word table style** (`table_style`, e.g. `Colorful List Accent 2`)
- **Automatic table column autofit** (`table_autofit`) so wide tables fit the page
- **Clean BLOB file output** — the document is returned as a proper Word blob that renders
  as a downloadable file card in workflow tool nodes.

Author: **polden**. Tool provider id: `polden/docx`.

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `markdown_content` | string (required) | — | Markdown to convert. |
| `title` | string | — | Optional document title (also used for the file name). |
| `margin_cm` | number | `1.5` | Uniform page margin in cm (0–5). Small = narrow margins. |
| `table_style` | select | `Table Grid` | Built-in Word table style, or `None` for the default borderless style. |
| `table_autofit` | boolean | `true` | Let Word auto-fit table column widths to content. |
| `delivery_mode` | select | `file` | `file` = return the DOCX as a blob (workflow file card). `link` = upload + return a download URL (Agent-safe, avoids `binary_link`). |

### Available table styles

`None`, `Table Grid`, `Light List Accent 1`, `Light Grid Accent 1`,
`Light Shading Accent 1`, `Medium Shading 1 Accent 1`, `Colorful List Accent 2`,
`Colorful Grid Accent 1`.

If a requested style cannot be applied, the plugin automatically falls back to
`Table Grid` and then to no style; the style actually used is reported in the
`table_style_applied` output.

## Outputs

| Output | Description |
|--------|-------------|
| (file) | The generated `.docx` as a BLOB → appears in the tool node `files` output / as a file card. |
| `filename` | The generated file name. |
| `table_style_applied` | The table style actually applied. |

## About the `binary_link` error

Some setups hit this when a DOCX tool is called **from an Agent node**:

```
tool invoke error: Failed to parse response: ... Input should be '... image_link ...'
[type=enum, input_value='binary_link', ...]
```

This is a Dify server/daemon-side issue (see `langgenius/dify-plugin-daemon#200`,
`langgenius/dify#36936`): when a non-image **BLOB** is streamed to an agent it is converted
to a `BINARY_LINK` message type that older daemons/SDKs do not accept. It is **not** caused
by this plugin's code.

What this plugin does about it — the **`delivery_mode`** parameter:

- **`file`** (default): returns a standard BLOB with the proper DOCX mime type. In a
  **workflow tool node** this produces a downloadable file card with no `binary_link`
  error. Do **not** use this mode when the tool is called from an **Agent node** on a Dify
  version that lacks the `BINARY_LINK` daemon fix (it will raise the parse error above).
- **`link`** (Agent-safe): the plugin uploads the generated file via the SDK
  (`session.file.upload`) and returns a **download URL** as a `link` + `text` message and a
  `file_url` variable. No BLOB is emitted, so Dify never converts it to `BINARY_LINK` — the
  Agent receives a clean, clickable download link instead of crashing.

**Rule of thumb:** Workflow tool node → `file`; Agent node → `link`.

The `binary_link` failure itself is a Dify server/daemon issue (see
`langgenius/dify-plugin-daemon#200`, `langgenius/dify#36936`), fixed upstream in newer
Dify releases. On a patched Dify, `file` mode works in agents too.

## Local development

```bash
pip install -r requirements.txt
cp .env.example .env      # fill REMOTE_INSTALL_KEY from Dify → Plugins → Debug
python -m main
```

## Packaging (.difypkg)

`manifest.yaml` must sit at the **root** of the archive.

```powershell
# from this plugin directory
Compress-Archive -Path * -DestinationPath ..\polden-docx.zip -Force
Rename-Item ..\polden-docx.zip polden-docx.difypkg
```

or with the Dify CLI:

```bash
dify plugin package ./
```

## Dependencies

- `dify_plugin` ≥ 0.9.1, < 1.0.0
- `python-docx`, `markdown`, `htmldocx`, `beautifulsoup4`

## License

MIT.
