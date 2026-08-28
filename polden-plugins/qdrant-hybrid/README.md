# Qdrant Hybrid Retriever (polden) for Dify

A focused Qdrant hybrid retrieval plugin for Dify, built on the official `qdrant-client`.
It runs dense + sparse (BM25) retrieval with configurable fusion (DBSF / RRF), optional
MMR diversification, optional tenant automode / weighted quotas, payload filters, and
optional reranking.

Version `1.0.6` adds optional `all_from_top_k_source`: after ranked retrieval, expand to
all points from the top-K sources (by descending sum of scores), without further
sort/rerank. `1.0.5` introduced independent `automode_select_db` / `tenants_ratios`.

## Highlights

- **Python client**: uses `qdrant_client.QdrantClient` and `models.query_points`.
- **Fusion**: `models.FusionQuery` with selectable `dbsf` (default) or `rrf`.
- **MMR**: optional `models.NearestQuery(mmr=models.Mmr(...))` diversification on the dense branch.
- **Sparse BM25**: `models.Document(text=..., model="Qdrant/bm25")` (local inference via `fastembed`).
- **score_threshold**: applied on the prefetch branches (native dense/sparse score scale).
- **Filters**: `point_type`, `corpus_docs`, `source`, and arbitrary `payload_filters` (JSON).
- **Tenant automode / ratios**: optional probe → median prune and/or quota merge across `corpus_docs`.
- **Rerank**: optional Dify rerank model (model-selector) with a built-in `BAAI/bge-reranker-base` fallback.
- **LLM-friendly outputs**: `result`, `source_nodes`, `source_list_text`, `chunk_texts`.

## Requirements

- Qdrant server 1.15+ (MMR requires 1.15+; tested against 1.18.x).
- A Dify embedding model (selected in the tool) for dense vectors.

## Credentials

- **Qdrant URL** (`base_url`): e.g. `https://xxx.cloud.qdrant.io:6333` or `http://localhost:6333`.
- **API Key** (`api_key`): optional (required for Qdrant Cloud).
- **Additional Headers** (`extra_headers`): optional JSON object of custom HTTP headers.

## Tool: Hybrid Retriever

Key parameters:

- `collection`, `text`, `embedding_model_config` (embedding model selector).
- `limit`, `prefetch_limit`.
- `fusion`: `dbsf` (default) or `rrf`.
- `score_threshold`: prefetch-level threshold (default `0.0`).
- `use_mmr` + `mmr_diversity`: enable/tune MMR diversification on the dense branch.
- `point_type`, `corpus_docs`, `source`: payload filters (`corpus_docs` = tenants list).
- `automode_select_db`: probe tenants and median-prune when `N > 5` (independent of ratios).
- `tenants_ratios`: allocate final top-K seats by amplified probe weights (quota merge).
- `probe_top_k`: hits per tenant during the scoring probe (effective `max(4, probe_top_k)`).
- `payload_filters`: arbitrary payload filter as a JSON object, e.g.
  `{"kind": ["Процедура"], "reg_area": ["Управление запасами"]}`.
- `enable_rerank`, `rerank_model_config`, `rerank_text_field`.
- `all_from_top_k_source`: if `>0`, after ranked results take top-K unique `source`
  values by descending sum of scores (`rerank_score` if present, else `score`), then
  scroll **all** matching points for those sources (keeps `point_type` / corpus /
  payload filters; no sort/rerank). Empty or `≤0` disables.
- `with_payload` (raw vectors are never returned).

### Tenant mode overrides (when `automode_select_db=true`)

| `# corpus_docs` | Effective behaviour |
|-----------------|---------------------|
| 1 | Legacy single query (no probe / no quotas) |
| 2…5 | No median prune; force `tenants_ratios` quota merge |
| >5 | Median prune; `tenants_ratios` follows the user flag |

Probe always uses `point_type=["chunk","summary"]`. Final retrieval uses the tool `point_type` parameter.

With both flags off (defaults), behaviour matches the previous single hybrid query.

### Outputs

- `result`: raw retrieved points (`id`, `score`, `payload`, optional `rerank_score`).
- `source_nodes`: cleaned nodes (`id`, `score`, `payload.text`, `payload.metadata`).
- `source_list_text`: numbered list of unique sources.
- `chunk_texts`: array of all chunk texts — usable directly as the Dify LLM node "context".

## Notes

- MMR operates inside the dense prefetch branch; fusion combines dense + sparse afterwards.
- If the hybrid query fails, the tool falls back to a dense-only query.
- Self-hosted Dify with signature verification enabled will reject unsigned packages
  (`FORCE_VERIFYING_SIGNATURE=false` or sign via `dify signature sign`).

## License

See [LICENSE](./LICENSE).
