# Network Calls Reference

This document lists every outbound network call made by `code-review-graph`, which data is
transmitted, and under what conditions each call occurs.  All cloud calls are **opt-in** and
**disabled by default**.  Normal operation (build, update, query) is fully offline.

---

## 1. Google Gemini Embeddings API

| Property | Value |
|----------|-------|
| **File** | `code_review_graph/embeddings.py` — `GoogleEmbeddingProvider.embed()` / `embed_query()` |
| **Trigger** | `provider="google"` passed to `get_provider()` **and** `GOOGLE_API_KEY` env var is set |
| **Endpoint** | Google Generative AI service via the `google-genai` SDK (HTTPS) |
| **Model** | `gemini-embedding-001` (configurable via `CRG_EMBEDDING_MODEL`) |

### Data sent to Google

**During graph indexing** (`embed()`):
```json
{
  "model": "gemini-embedding-001",
  "contents": ["<symbol text>", "..."],
  "config": { "task_type": "RETRIEVAL_DOCUMENT" }
}
```

**During semantic search** (`embed_query()`):
```json
{
  "model": "gemini-embedding-001",
  "contents": ["<query string>"],
  "config": { "task_type": "RETRIEVAL_QUERY" }
}
```

The `<symbol text>` is a short, whitespace-joined string built from public AST metadata only
(see `_node_to_text()`).  **Source code bodies are never included.**  Example:
```
authenticate function in UserService (username: str, password: str) returns bool python
```

Batches of up to 100 symbols are sent per request.

### Authentication
`Authorization` header is set by the Google SDK using the value of `GOOGLE_API_KEY`.

### Privacy note
Symbol names, parameter signatures, return types, and programming language are sent to
Google's servers.  Review [Google's privacy policy](https://policies.google.com/privacy) before
enabling this provider in projects where symbol names are confidential.

---

## 2. MiniMax Embeddings API

| Property | Value |
|----------|-------|
| **File** | `code_review_graph/embeddings.py` — `MiniMaxEmbeddingProvider._call_api()` |
| **Trigger** | `provider="minimax"` passed to `get_provider()` **and** `MINIMAX_API_KEY` env var is set |
| **Endpoint** | `POST https://api.minimax.io/v1/embeddings` |
| **Model** | `embo-01` (1536 dimensions, fixed) |

### Data sent to MiniMax

**During graph indexing** (task type `"db"`):
```json
{
  "model": "embo-01",
  "texts": ["<symbol text>", "..."],
  "type": "db"
}
```

**During semantic search** (task type `"query"`):
```json
{
  "model": "embo-01",
  "texts": ["<query string>"],
  "type": "query"
}
```

`<symbol text>` follows the same format as the Google provider (public AST metadata only, no
source code bodies).  Batches of up to 100 symbols are sent per request.

### Authentication
```
Authorization: Bearer <MINIMAX_API_KEY>
Content-Type: application/json
```

### Privacy note
Symbol names, parameter signatures, return types, and programming language are sent to
MiniMax's servers.  Review [MiniMax's privacy policy](https://www.minimax.io/privacy-policy)
before enabling this provider in projects where symbol names are confidential.

---

## 3. HuggingFace Hub — local model download (sentence-transformers)

| Property | Value |
|----------|-------|
| **File** | `code_review_graph/embeddings.py` — `LocalEmbeddingProvider._get_model()` |
| **Trigger** | `provider="local"` (the default) **and** the `[embeddings]` extra is installed |
| **Endpoint** | `https://huggingface.co` (HTTPS, managed by the `sentence-transformers` library) |
| **Frequency** | **One-time** download; model weights are cached locally by the library |

### Data sent to HuggingFace

A standard HTTPS `GET` request to download the model weights.  **No user source-code data or
symbol text is ever sent.**  The only information transmitted is the model identifier and the
requester's IP address, which is the same as any package download.

Default model: `all-MiniLM-L6-v2` (configurable via the `CRG_EMBEDDING_MODEL` env var or the
`model` argument to `get_provider()`).  After the one-time download, all embedding computation
runs locally — no further network traffic occurs.

---

## 4. D3.js CDN — browser-side visualization

| Property | Value |
|----------|-------|
| **File** | `code_review_graph/visualization.py` (two `<script>` tags in the generated HTML) |
| **Trigger** | User opens a generated `graph.html` file in a browser |
| **Endpoint** | `https://d3js.org/d3.v7.min.js` (CDN, browser `GET` request) |
| **Data sent** | None — standard browser request for a static JavaScript file |

The script tag includes a [Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)
hash to guard against CDN tampering:
```html
<script src="https://d3js.org/d3.v7.min.js"
        integrity="sha384-CjloA8y00+1SDAUkjs099PVfnY2KmDC2BZnws9kh8D/lX1s46w6EPhpXdqMfjK6i"
        crossorigin="anonymous"></script>
```

**No graph data or source-code information is sent to d3js.org.**  If offline use is required,
replace the CDN `<script>` tag with a local copy of D3.js.

---

## 5. Evaluation benchmark — git clone / git fetch

| Property | Value |
|----------|-------|
| **File** | `code_review_graph/eval/runner.py` — `clone_or_update()` |
| **Trigger** | Running `code-review-graph eval` (evaluation / CI benchmarking only) |
| **Protocol** | `git clone` / `git fetch` over HTTPS (GitHub) |
| **Data sent** | None — read-only clone of public open-source repositories |

The eval pipeline downloads a set of well-known open-source repositories (FastAPI, Flask,
Express, etc.) as defined in `code_review_graph/eval/configs/*.yaml`.  No user source-code is
transmitted; this call is only ever made when running benchmarks and has no effect during normal
MCP server operation.

---

## Summary table

| Call | Default | When active | User data sent |
|------|---------|-------------|----------------|
| Google Gemini embeddings | **Off** | `provider="google"` + `GOOGLE_API_KEY` | Symbol names, signatures, return types (no source bodies) |
| MiniMax embeddings | **Off** | `provider="minimax"` + `MINIMAX_API_KEY` | Symbol names, signatures, return types (no source bodies) |
| HuggingFace model download | **Off** | `[embeddings]` extra installed, first run | None (model weights only) |
| D3.js CDN | **Off** | User opens `graph.html` in browser | None |
| Eval git clone | **Off** | `code-review-graph eval` command | None (reads public repos) |
