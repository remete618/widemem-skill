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
- Returns: `{count, confidence, memories: [{id, content, importance, score}]}`

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

| Level | Similarity | Meaning |
|-------|-----------|---------|
| HIGH | >= 0.65 | Strong semantic match |
| MODERATE | >= 0.45 | Decent match, some uncertainty |
| LOW | >= 0.25 | Weak match, may be irrelevant |
| NONE | < 0.25 | No relevant results found |

## Setup

Add to `.mcp.json` (project or `~/.claude/.mcp.json` for global):

```json
{
  "mcpServers": {
    "widemem": {
      "command": "python",
      "args": ["-m", "widemem.mcp_server"],
      "env": {
        "WIDEMEM_LLM_PROVIDER": "ollama",
        "WIDEMEM_LLM_MODEL": "llama3.2",
        "WIDEMEM_EMBEDDING_PROVIDER": "sentence-transformers"
      }
    }
  }
}
```

Install: `pip install widemem-ai[mcp]`

For OpenAI: set `WIDEMEM_LLM_PROVIDER=openai` and `OPENAI_API_KEY` env var.
For Anthropic: set `WIDEMEM_LLM_PROVIDER=anthropic` and `ANTHROPIC_API_KEY`.
