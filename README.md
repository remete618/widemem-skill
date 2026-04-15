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
