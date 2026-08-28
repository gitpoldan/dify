# RWs3 — S3 Read/Write Plugin

Dify plugin for simple S3/MinIO operations: **list**, **read**, **write**, **delete**.

## Tools

| Tool | Description |
|------|-------------|
| `s3_list` | List objects under a prefix |
| `s3_read` | Read file content (UTF-8, with truncation limit) |
| `s3_write` | Write text content |
| `s3_delete` | Delete object (idempotent) |

All tools return structured JSON: `{ "ok": true, "data": ... }` or `{ "ok": false, "error": ... }`.

## Provider credentials

Configure your S3/MinIO connection in the Dify tool provider:

- **endpoint** — MinIO host (empty for AWS S3)
- **access_key_id** / **secret_access_key**
- **use_https** — default `true`
- **bucket** — required
- **region** — default `us-east-1`
- **prefix** — optional namespace guard (all keys resolved under this prefix)
- **max_read_bytes** / **max_write_bytes** — size limits

## Local development

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
pip install pytest
pytest tests/
python -m main
```

## Packaging

```bash
python package.py
```

Produces `../polden-RWs3_0.0.1.difypkg`.
