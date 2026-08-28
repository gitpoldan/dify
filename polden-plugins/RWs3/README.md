# RWs3 — S3 Read/Write Plugin

Simple, reliable Dify tool plugin for **S3** and **MinIO** object storage:
list directory contents, read files, write text files, and delete objects.

Author: **gitpoldan**. Tool provider id: `gitpoldan/rws3`.

## Why this plugin

- **AWS S3 and MinIO** from one provider — path-style addressing for MinIO is handled automatically.
- **Optional prefix guard** — restrict all operations to a namespace inside your bucket.
- **Structured JSON responses** — every tool returns `{ "ok": true, "data": ... }` or `{ "ok": false, "error": ... }`, friendly for LLM agents and workflows.
- **Size limits** — configurable read/write byte limits to protect agent context windows.

## Tools

| Tool | Description |
|------|-------------|
| `s3_list` | List objects under a prefix (directory listing) |
| `s3_read` | Read file content as UTF-8 text (with truncation limit) |
| `s3_write` | Write UTF-8 text content to an object |
| `s3_delete` | Delete an object (idempotent) |

### s3_list

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prefix` | string | No | Relative directory prefix inside the configured base prefix |

**Success response (`data`):**

```json
{
  "prefix": "my-app/docs/",
  "items": [
    { "key": "my-app/docs/readme.txt", "size": 128, "etag": "abc...", "last_modified": "2026-08-28T08:00:00+00:00" }
  ],
  "count": 1
}
```

### s3_read

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | string | Yes | Object key relative to the configured base prefix |
| `max_bytes` | number | No | Max bytes to return (overrides provider default) |

**Success response (`data`):**

```json
{
  "key": "my-app/docs/readme.txt",
  "content": "Hello, S3!",
  "size": 10,
  "sha256": "...",
  "truncated": false,
  "encoding": "utf-8"
}
```

**Error response (not found):**

```json
{
  "ok": false,
  "error": { "code": "not_found", "message": "Object 'docs/missing.txt' not found", "retryable": false }
}
```

### s3_write

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | string | Yes | Object key relative to the configured base prefix |
| `content` | string | Yes | UTF-8 text to write |
| `content_type` | string | No | MIME type (default `text/plain; charset=utf-8`) |

**Success response (`data`):**

```json
{
  "key": "my-app/docs/readme.txt",
  "size": 10,
  "sha256": "...",
  "etag": "..."
}
```

### s3_delete

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | string | Yes | Object key relative to the configured base prefix |

**Success response (`data`):**

```json
{ "key": "my-app/docs/readme.txt", "deleted": true }
```

Deleting a non-existent key still returns success (idempotent).

## Provider credentials

Configure in the Dify tool provider settings:

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `endpoint` | No | empty (AWS S3) | MinIO host, e.g. `minio.example.com:9000` |
| `access_key_id` | Yes | — | S3 access key |
| `secret_access_key` | Yes | — | S3 secret key |
| `use_https` | No | `true` | Use HTTPS for custom endpoint |
| `bucket` | Yes | — | Bucket name |
| `region` | No | `us-east-1` | AWS region |
| `prefix` | No | empty | Base prefix — all tool keys are resolved under this path |
| `max_read_bytes` | No | `204800` | Max bytes returned by `s3_read` |
| `max_write_bytes` | No | `5242880` | Max bytes accepted by `s3_write` |

On save, the provider validates credentials with `head_bucket()`.

## Limitations

- **Text only** — `s3_write` accepts UTF-8 strings; binary/base64 is not supported in v0.0.1.
- **Read truncation** — large files are truncated to `max_read_bytes`; check `truncated: true` in the response.
- **Prefix guard** — when `prefix` is set, tool keys cannot escape that namespace.
- **Key validation** — keys must not contain `..`, leading `/`, or backslashes.

## Local development

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
pip install -r requirements.txt
pip install pytest
pytest tests/
python -m main                # remote debug — set REMOTE_INSTALL_* in .env if needed
```

## Packaging (.difypkg)

`manifest.yaml` must sit at the **root** of the archive.

```bash
# Official Dify CLI (recommended for Marketplace submission)
dify plugin package ./

# Or custom script (Unix-style zip paths)
python package.py
```

Produces `../gitpoldan-rws3_0.0.1.difypkg`.

## Dependencies

- `dify_plugin` >= 0.9.0
- `boto3` >= 1.35.0

## License

MIT. See [LICENSE](LICENSE).

## Support

- Source: https://github.com/gitpoldan/dify/tree/main/polden-plugins/RWs3
- GitHub Issues: https://github.com/gitpoldan/dify/issues
- Email: bv2020donch@gmail.com

Russian documentation: [readme/README_ru_RU.md](readme/README_ru_RU.md).
