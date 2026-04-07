# Legal & Privacy

**License:** MIT (see [LICENSE](../LICENSE) in project root)

**Privacy:**
- Zero telemetry
- All graph data stored locally in `.code-review-graph/graph.db`
- No network calls during normal operation (build, update, query)
- Optional embeddings model downloaded once from HuggingFace when using the `[embeddings]` extra
  (model weights only; no user data is sent)
- Optional cloud embedding providers (Google Gemini, MiniMax) transmit symbol names and
  signatures — but never source-code bodies — when explicitly configured with an API key
- See [`network-calls.md`](network-calls.md) for a full list of every outbound call and the
  exact data transmitted

**Data:** Source-code bodies never leave your machine.  When a cloud embedding provider is
enabled, minimal AST metadata (symbol names, parameter signatures, return types) is sent to
the configured provider's API.

**Warranty:** Provided as-is, without warranty of any kind.
