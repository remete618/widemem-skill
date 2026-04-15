# widemem MCP Tool Reference

## Tools

### `widemem_add` — Store memories
Extract and store facts from text. Handles deduplication and conflict resolution automatically.
- `text` (required, string): The text to extract facts from
- `user_id` (optional, string): Isolate memories per user
- Returns: `{added, memories: [{id, content, importance}], clarifications?}`

### `widemem_search` — Semantic search
Returns memories ranked by similarity, importance, and recency. Includes confidence level.
- `query` (required, string): Natural language search query
- `user_id` (optional, string): Filter to specific user
- `top_k` (optional, int, default 5, max 100): Number of results
- Returns: `{count, confidence, memories: [{id, content, importance, score, created_at}]}`

### `widemem_pin` — Pin critical facts
Store with elevated importance (9.0). Use for facts the user explicitly wants remembered, corrections to forgotten info, or YMYL data.
- `text` (required, string): The fact to pin
- `user_id` (optional, string): Isolate per user
- Returns: `{pinned, memories: [{id, content, importance}]}`

### `widemem_delete` — Remove a memory
- `memory_id` (required, string): UUID of the memory to delete
- Returns: `{deleted: <id>}`

### `widemem_count` — Count stored memories
- `user_id` (optional, string): Filter to specific user
- Returns: `{count: <n>}`

### `widemem_export` — Export all memories
Export as JSON array. Optionally filter by user.
- `user_id` (optional, string): Filter to specific user
- Returns: JSON string of all memories

### `widemem_health` — Health check
- Returns: `{status: "ok"}`

## Confidence Thresholds

Default thresholds (tuned for `sentence-transformers/all-MiniLM-L6-v2`, configurable via environment variables):

| Level | Default | Env Variable | Meaning |
|-------|---------|-------------|---------|
| HIGH | >= 0.45 | `WIDEMEM_CONFIDENCE_HIGH` | Strong semantic match |
| MODERATE | >= 0.25 | `WIDEMEM_CONFIDENCE_MODERATE` | Decent match, some uncertainty |
| LOW | >= 0.12 | `WIDEMEM_CONFIDENCE_LOW` | Weak match, may be irrelevant |
| NONE | < lowest | — | No relevant results found |

These thresholds are calibrated for `all-MiniLM-L6-v2`, which produces scores in the 0.1-0.6 range for typical queries. If you switch to OpenAI embeddings, raise the thresholds (e.g., HIGH=0.65, MODERATE=0.45, LOW=0.25).

### Search response fields

Each memory in search results includes:
- `id` — UUID for deletion/reference
- `content` — the stored fact text
- `importance` — 1-10 importance score
- `score` — combined similarity/importance/recency score
- `created_at` — ISO timestamp of when the memory was stored (use for conflict resolution)

## Setup

Add to `.mcp.json` (project or `~/.claude/.mcp.json` for global):

```json
{
  "mcpServers": {
    "widemem": {
      "command": "python3",
      "args": ["-m", "widemem.mcp_server"],
      "env": {
        "WIDEMEM_LLM_PROVIDER": "openai",
        "WIDEMEM_LLM_MODEL": "gpt-4o-mini",
        "WIDEMEM_EMBEDDING_PROVIDER": "sentence-transformers",
        "WIDEMEM_CONFIDENCE_HIGH": "0.45",
        "WIDEMEM_CONFIDENCE_MODERATE": "0.25",
        "WIDEMEM_CONFIDENCE_LOW": "0.12",
        "OPENAI_API_KEY": "your-key-here"
      }
    }
  }
}
```

Install: `pip install widemem-ai[mcp,sentence-transformers]`

**Local-only setup (advanced):** To run without an API key, use Ollama instead. Requires `ollama serve` running with a model pulled (8B+ recommended for reliable conflict resolution):
```
"WIDEMEM_LLM_PROVIDER": "ollama",
"WIDEMEM_LLM_MODEL": "llama3",
```
Remove the `OPENAI_API_KEY` line. Note: 3B models (llama3.2) produce JSON errors and unreliable conflict resolution.
