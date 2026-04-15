# widemem — Claude Code Memory Skill

**Remembers things at scale and knows when it doesn't.**

Claude's built-in memory works fine for 20 facts. When you have hundreds of memories across users and projects, you need semantic search, importance scoring, and confidence levels — not grep over a markdown file.

widemem gives Claude Code a real memory engine: vector embeddings, LLM-based fact extraction, temporal decay, and the honesty to say "I don't have that" instead of guessing.

Built on [widemem-ai](https://github.com/remete618/widemem-ai) — the open-source memory layer for AI agents.

## Built-in memory vs widemem

| | Claude's MEMORY.md | widemem |
|---|---|---|
| Storage | Flat markdown | Vector embeddings (FAISS/Qdrant) |
| Search | Grep / full context load | Semantic similarity |
| Knows when it doesn't know | Limited | Yes — 4 confidence levels |
| Importance scoring | No | 1-10 with temporal decay |
| Dedup / conflict detection | No | LLM-based |
| Quality gates | No | 5-tier classification |
| Scales to 1000s of facts | Poorly (context window) | Yes (vector index) |
| Multi-user support | No | Built-in |

## With this skill

```
You: "What city do I live in?"
AI:  confidence: NONE — "I don't have that stored. Want to tell me?"
You: "Berlin."
AI:  [stores with importance 7, semantic embedding, never guesses again]

...500 memories later...

You: "What do you know about my work?"
AI:  confidence: HIGH — "You're a data scientist at Stripe.
     You prefer Python with type hints. You're on the platform team."
```

## Install

### 1. Install widemem

```bash
pip install widemem-ai[mcp,sentence-transformers]
```

For OpenAI embeddings instead of local:
```bash
pip install widemem-ai[mcp]
```

### 2. Install the skill

**Option A — Plugin (recommended):**
```bash
# In Claude Code
/plugin install widemem
```

**Option B — Manual:**
```bash
# Copy skill files to your project
git clone https://github.com/remete618/widemem-skill.git
mkdir -p .claude/skills
cp -r widemem-skill/skills/widemem .claude/skills/

# Add MCP config (or merge into your existing .mcp.json)
cp widemem-skill/.mcp.json .mcp.json
```

### 3. Configure

Add your OpenAI API key to `.mcp.json` (replace `your-key-here`). The default config uses `gpt-4o-mini` for fact extraction and conflict resolution (~$0.01/operation).

| Env Variable | Default | Options |
|---|---|---|
| `WIDEMEM_LLM_PROVIDER` | `openai` | `openai`, `anthropic`, `ollama` |
| `WIDEMEM_LLM_MODEL` | `gpt-4o-mini` | Any model your provider supports |
| `WIDEMEM_EMBEDDING_PROVIDER` | `sentence-transformers` | `openai`, `sentence-transformers` |
| `WIDEMEM_DATA_PATH` | `~/.widemem/data` | Any local path |
| `WIDEMEM_CONFIDENCE_HIGH` | `0.45` | Similarity threshold for HIGH confidence |
| `WIDEMEM_CONFIDENCE_MODERATE` | `0.25` | Similarity threshold for MODERATE confidence |
| `WIDEMEM_CONFIDENCE_LOW` | `0.12` | Similarity threshold for LOW confidence |

**Local-only setup (advanced):** To run without an API key, set `WIDEMEM_LLM_PROVIDER=ollama` and `WIDEMEM_LLM_MODEL=llama3` (8B+ recommended). 3B models produce JSON errors and unreliable conflict resolution. Remove the `OPENAI_API_KEY` line.

## Commands

| Command | What it does |
|---------|-------------|
| `/mem search <query>` | Semantic search across all memories |
| `/mem add <text>` | Store a fact (with quality gates — not everything gets in) |
| `/mem pin <text>` | Pin critical fact at importance 9.0 (allergies, API keys, things that matter) |
| `/mem stats` | Memory count and health check |
| `/mem export` | Export all memories as JSON |
| `/mem import` | Import facts from Claude's MEMORY.md into widemem |
| `/mem reflect` | Full memory audit — find contradictions, duplicates, staleness |

Or just talk normally. The skill triggers on "remember this", "do you remember", "what do you know about me", and similar phrases.

## The "Not Everything Gets Stored" Problem

Most memory tools store everything. You say "ok thanks" and now that's a memory forever. You sneeze and it stores your sneeze.

widemem has quality gates. Every incoming fact gets classified before storage:

| Classification | Example | What happens |
|---|---|---|
| **CRITICAL** | "Remember: I'm allergic to penicillin" | Pinned at importance 9.0 |
| **HIGH** | "Actually I moved to Berlin" | Deletes old location, stores new |
| **MEDIUM** | "I work at Stripe" | SDK handles dedup, then store |
| **LOW** | "I'm on the dev branch right now" | Stored only if you insist |
| **SKIP** | "ok", "thanks", "run the tests" | Silently ignored |

Your memory stays clean. Your vector store stays relevant. Your facts stay accurate.

## Confidence Scoring

Every search tells you how sure it is:

| Level | Similarity | What the AI says |
|---|---|---|
| **HIGH** | >= 0.45 | Answers confidently |
| **MODERATE** | >= 0.25 | "I think so, but not 100% sure" |
| **LOW** | >= 0.12 | "I have a vague recollection..." |
| **NONE** | < 0.12 | "I don't have that stored." |

**The skill never makes up memories.** If confidence is NONE, it says so instead of guessing.

## Frustration Detection

```
You: "I ALREADY told you my anniversary is March 15th"
```

The skill detects frustration, searches harder, and when you repeat the fact, pins it at high importance so it never misses it again. Turns out user annoyance is actually useful data.

## Memory Reflection

`/mem reflect` runs a full audit:

- **Duplicates** — "Lives in SF" and "User's city is San Francisco" → keep one
- **Contradictions** — "Works at Google" vs "Works at Meta" → which is current?
- **Stale entries** — "Debugging auth module" from 3 weeks ago → still relevant?
- **Consolidation** — "Likes Python" + "Uses mypy" + "Prefers typed code" → merge

Reports everything before touching anything. Nothing gets deleted without your say-so.

## Migrating from MEMORY.md

Already have facts in Claude's built-in memory? Import them:

```
/mem import
```

The skill reads your MEMORY.md index, follows each linked file, extracts the facts, runs quality gates, and stores them in widemem. Duplicates are skipped. Your original MEMORY.md files are not deleted.

## How It Works

```
You: "I work at Stripe as a data scientist"
     |
     v
Quality Gate: MEDIUM (personal fact) --> call widemem_add
     |
     v
widemem_add --> LLM extracts fact, dedup check, conflict resolution --> store (importance: 7)
     |
     v
Stored in FAISS with embedding vector

...three weeks later...

You: "What do you know about my job?"
     |
     v
widemem_search --> confidence: HIGH, similarity: 0.89
     |
     v
"I recall you work at Stripe as a data scientist."
```

## Architecture

```
+----------------------------------+
|  Claude Code + /mem skill        |
|  (SKILL.md + references/)        |
+-----------+----------------------+
            | MCP protocol (stdio)
+-----------v----------------------+
|  widemem MCP Server              |
|  7 tools: add, search, pin,     |
|  delete, count, export, health   |
+-----------+----------------------+
            |
+-----------v----------------------+
|  widemem-ai SDK                  |
|  Fact extraction, conflict       |
|  resolution, confidence scoring  |
+----------+-----------+-----------+
| FAISS/   | SQLite    | LLM       |
| Qdrant   | history   | (any)     |
+----------+-----------+-----------+
```

## vs. Other Memory Solutions

| Feature | widemem | claude-mem | Pure SKILL.md skills |
|---------|---------|-----------|---------------------|
| Semantic search | FAISS/Qdrant | Chroma | grep |
| Confidence scoring | 4 levels | No | No |
| Quality gates | 5 tiers | Basic | Some |
| Frustration detection | Yes | No | No |
| YMYL protection | Yes | No | No |
| Conflict resolution | LLM-based | No | No |
| Memory pinning | Yes | No | No |
| Self-reflection | `/mem reflect` | No | `/memory prune` |
| Runs locally | Embeddings local, LLM cloud (or Ollama) | Requires Chroma + Bun | Filesystem |
| Install | pip + skill | npm + plugin | curl |
| Says "I don't know" | Yes (4 confidence levels) | Basic | No |

## Requirements

- Python 3.10+
- `widemem-ai[mcp,sentence-transformers]` (`pip install widemem-ai[mcp,sentence-transformers]`)
- OpenAI API key (default) or Anthropic API key
- For fully local operation (advanced): Ollama with an 8B+ model instead of an API key

## Cost

widemem uses an LLM for fact extraction and conflict resolution. Search is free (local embeddings).

### What costs money

| Operation | What happens | Cost (gpt-4o-mini) |
|---|---|---|
| `widemem_add` | LLM extracts facts, checks for conflicts | ~$0.0001 |
| `widemem_pin` | Same as add, with elevated importance | ~$0.0001 |
| `widemem_search` | Local embeddings only, no LLM call | **$0** |
| `/mem reflect` | LLM analyzes exported memories for issues | ~$0.001 |

### Monthly estimates

| Usage level | Description | Monthly cost |
|---|---|---|
| **Light** | Casual coding, ~10 stores/day | ~$0.05 |
| **Moderate** | Regular coding, ~20 stores/day, weekly reflect | ~$0.15 |
| **Heavy** | All-day coding, ~50 stores/day, frequent reflects | ~$0.40 |
| **Extreme** | 200 stores/day, 10+ hours/day | ~$1.50 |

For context: a single Claude Code conversation with Opus costs more than a month of heavy widemem usage. The ceiling is low because gpt-4o-mini is $0.15/1M input tokens, each operation sends ~500 tokens, and search (the most frequent operation) is free.

### Fully local (zero cost)

With Ollama, everything runs on your machine. No API calls, no cost, no data leaves your laptop. The tradeoff: local models (especially 3B) are less reliable at fact extraction and conflict resolution. Use 8B+ for decent results. See "Bring Your Own LLM" below.

## Bring Your Own LLM

widemem separates the embedding model (for search) from the LLM (for fact extraction and conflict resolution). You can mix and match.

### Tested configurations

| LLM | Extraction quality | Conflict resolution | JSON reliability | Speed |
|---|---|---|---|---|
| **gpt-4o** | Best | Works (replaces old facts) | Perfect | 19s/6 facts |
| **gpt-4o-mini** (default) | Good | Works (replaces old facts) | Perfect | 25s/6 facts |
| **llama3 8B** (Ollama) | Good | Fails (keeps both old and new) | Good | 89s/6 facts |
| **llama3.2 3B** (Ollama) | Poor (drops facts) | Fails | Unreliable (JSON errors) | 48s/4 facts |

### How to switch

Edit the `env` block in your `.mcp.json`:

**OpenAI (default, recommended):**
```json
"WIDEMEM_LLM_PROVIDER": "openai",
"WIDEMEM_LLM_MODEL": "gpt-4o-mini",
"OPENAI_API_KEY": "sk-..."
```

**Anthropic:**
```json
"WIDEMEM_LLM_PROVIDER": "anthropic",
"WIDEMEM_LLM_MODEL": "claude-haiku-4-5-20251001",
"ANTHROPIC_API_KEY": "sk-ant-..."
```

**Ollama (local, free):**
```json
"WIDEMEM_LLM_PROVIDER": "ollama",
"WIDEMEM_LLM_MODEL": "llama3"
```
Requires `ollama serve` running. Use `ollama pull llama3` first. Remove the API key line.

### What the LLM does (and doesn't do)

The LLM handles two jobs:
1. **Fact extraction** -- turns "I work at Stripe as a backend engineer" into a clean fact with an importance score
2. **Conflict resolution** -- decides whether a new fact contradicts, duplicates, or extends an existing memory

The LLM does NOT handle search. Search uses the embedding model (sentence-transformers by default), which runs locally and is free. So even with a cloud LLM, your searches never leave your machine and never cost money.

### Tuning confidence thresholds per embedding model

The default thresholds (0.45/0.25/0.12) are calibrated for `sentence-transformers/all-MiniLM-L6-v2`. If you switch to OpenAI embeddings (`text-embedding-3-small`), scores run higher and you should raise the thresholds:

| Embedding provider | HIGH | MODERATE | LOW |
|---|---|---|---|
| sentence-transformers (default) | 0.45 | 0.25 | 0.12 |
| OpenAI embeddings | 0.65 | 0.45 | 0.25 |

Set via `WIDEMEM_CONFIDENCE_HIGH`, `WIDEMEM_CONFIDENCE_MODERATE`, `WIDEMEM_CONFIDENCE_LOW` in `.mcp.json`.

## The Full SDK

This skill is a taste of [widemem-ai](https://github.com/remete618/widemem-ai). The Python SDK can do more:

- Hierarchical memory (facts → summaries → themes)
- Retrieval modes (fast/balanced/deep)
- Topic weighting and custom extraction
- YMYL protection for health/financial/legal data
- Time-range queries and temporal decay functions
- Batch conflict resolution
- Export/import, history audit trails

If the skill makes you want more, the SDK is where it lives.

## License

Apache 2.0 — same as [widemem-ai](https://github.com/remete618/widemem-ai).
