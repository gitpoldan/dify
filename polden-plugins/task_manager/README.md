# Transit Task Manager (polden)

A multi-tenant "transit" task manager for Dify Workflow and Agent apps. Tasks are
stored in a **MinIO / S3 bucket**, one SQLite database per tenant
(`{prefix}/{namespace?}/{tenant}.sqlite`). The plugin exposes a small task API as
Dify tools: create / read / list / update / status-transition / delete, plus TTL
auto-expiration and per-tenant statistics.

## Why this design

- **No own web server.** Dify's plugin runtime is the transport; the tools *are*
  the API that Workflow/Agent nodes call. FastAPI is unnecessary.
- **SQLModel over SQLite, per tenant.** Each tool call pulls the tenant DB from
  S3 into a temp file, runs SQLModel operations, and (for writes) uploads it back.
- **Safe concurrency.** Writes are serialized with a lease lock object
  (`{tenant}.lock`) and additionally guarded by an ETag precondition
  (`IfMatch` / `IfNoneMatch`) on the database object, retrying on conflict.
- **Lazy TTL.** Plugins have no scheduler, so a task whose TTL passed is
  auto-expired on the next access (`get_task` / `list_tasks` / `get_stats`) or in
  bulk via `sweep_ttl`.

## Storage backend

This plugin needs its own S3 / MinIO connection. Dify does not let one plugin
reuse another plugin's provider, so the credentials mirror the shape of the
`shenfor/minio_s3_storage` datasource plugin (`endpoint`, `access_key_id`,
`secret_access_key`, `use_https`) plus `bucket`, `region`, `prefix`. Point them at
the same MinIO server to keep everything in one place.

The bucket must already exist. Databases are created automatically on first write.

## Status model

`created` -> `pending` -> `in_progress` -> `completed` / `cancelled` / `failed` /
`expired`. Transitions are validated by a state machine (override with
`force=true`). Every transition can carry a JSON `metadata` object, recorded on
the task and in an append-only history table.

## Tools

| Tool | Purpose |
| --- | --- |
| `create_task` | Create a task (optional TTL, payload, metadata). |
| `get_task` | Fetch one task by id; evaluates TTL. |
| `list_tasks` | List/filter tasks by status with paging. |
| `update_task` | Patch data fields without changing status. |
| `set_status` | Transition status with metadata (state machine). |
| `delete_task` | Hard-delete, or soft-delete (cancel). |
| `sweep_ttl` | Bulk-expire stale tasks (supports `dry_run`). |
| `get_stats` | Task counts per status for a tenant. |

Every call requires `tenant`. An optional `namespace` groups several tenant
databases under a logical sub-path.

## Concurrency note

Conditional writes require an S3-compatible store that supports `If-Match` /
`If-None-Match` on `PutObject` (AWS S3 and recent MinIO releases do). Without
them the lease lock still serializes writers, but the ETag safety net is a no-op.
