---
name: mem
description: Persistent AI memory layer. Use when the user asks to remember something, recall past context, store facts, search memories, forget something, or manage long-term memory. Triggers on "do you remember", "I told you", "save this", "what do you know about me", "forget this", "memory stats", "export memories", or any /mem subcommand.
argument-hint: [search|add|stats|export|import|pin|reflect] [args...]
allowed-tools: mcp__widemem__widemem_add, mcp__widemem__widemem_search, mcp__widemem__widemem_delete, mcp__widemem__widemem_count, mcp__widemem__widemem_health, mcp__widemem__widemem_pin, mcp__widemem__widemem_export, Bash, Agent
---

# widemem — Persistent Memory for Claude Code

You have access to widemem, a persistent memory layer that stores, retrieves, and manages facts across conversations using semantic search, importance scoring, and temporal decay.

## Memory State
Count: !`python3 -c "from widemem.core.memory import WideMemory; print(WideMemory().count())" 2>/dev/null || echo "N/A"`

## Slash Commands

When invoked as `/mem`, parse `$ARGUMENTS` to determine the subcommand:

| Command | Action |
|---------|--------|
| `/mem search <query>` | Search memories via `widemem_search`. Show confidence level. |
| `/mem add <text>` | Run quality gates, then store via `widemem_add`. |
| `/mem pin <text>` | Pin critical fact with high importance via `widemem_pin`. |
| `/mem stats` | Call `widemem_count` + `widemem_health`, show summary. |
| `/mem export` | Export all memories as JSON via `widemem_export`. |
| `/mem import` | Import from MEMORY.md into widemem. See below. |
| `/mem reflect` | Full memory audit. See references/reflection-guide.md. |
| `/mem <anything else>` | Treat as a search query. |

## Quality Gates — Before Storing

Before every `widemem_add` call, evaluate the content:

1. **Classify:**
   - SKIP: greetings, "ok", "thanks", typos, tool output, ephemeral commands
   - LOW: trivial preferences, one-time context — store only if explicitly asked
   - MEDIUM: stated preferences, project facts, relationships — store
   - HIGH: corrections to existing memories, repeated info — store immediately
   - CRITICAL: user said "remember this" / "don't forget" — use `widemem_pin`

2. **If SKIP** — do not store, proceed silently.
3. **If LOW** — skip unless user explicitly asks to store.
4. **If MEDIUM+** — call `widemem_add` directly. The SDK handles deduplication and conflict resolution internally (LLM-based extraction, vector similarity check, and batch conflict resolution). Do NOT search before storing; that duplicates work the SDK already does.
5. **If CRITICAL** — use `widemem_pin` instead of `widemem_add`.

If `widemem_add` returns a `clarifications` field, surface it to the user for resolution.

See references/quality-gates.md for detailed criteria and examples.

## Confidence Handling

Search results include a `confidence` field returned by `widemem_search`. Use that field to determine your response style. Do not assess confidence yourself.

| Confidence | Action |
|------------|--------|
| **HIGH** | Answer confidently from memory |
| **MODERATE** | Answer, but note uncertainty |
| **LOW** | Mention vague recollection, offer to search deeper |
| **NONE** | Say you don't have it. Offer to save if user provides it. |

**Never fabricate memories.** If confidence is LOW or NONE, say so. If the returned memory content seems inconsistent with recent conversation context, note that even if confidence is HIGH.

## Frustration Detection

If the user says "I already told you this" or "you should know this":
1. Search with a broader query and higher top_k
2. If still not found — apologize and ask them to repeat
3. When they do — pin it immediately via `widemem_pin`

## Background vs Foreground

**Foreground** (user is waiting): `/mem` commands, "do you remember", direct memory questions.

**Background** (non-blocking): When the user shares facts mid-conversation while working on something else. Spawn a background agent to run quality gates and store if appropriate — don't interrupt their workflow.

## Behavior Guidelines

1. **Proactive storage**: When users share facts about themselves, store without being asked (after quality gates).
2. **Proactive retrieval**: Before answering personal questions, search widemem first.
3. **Transparency**: Say "I recall you mentioned..." when using stored memories.
4. **Respect deletion**: When asked to forget, delete and confirm.

## Import from MEMORY.md

When the user runs `/mem import`:

1. Find the MEMORY.md file. Use Bash: `find ~/.claude/projects -name "MEMORY.md" -path "*/memory/*" 2>/dev/null`
   - If multiple found, show the list and ask the user which to import
   - If none found, ask the user for the path
2. For each entry in the index, read the linked `.md` file to get the full content
3. For each memory file:
   - Extract the core facts from the content
   - Run quality gates on each fact (most will be MEDIUM or higher)
   - Store via `widemem_add`
4. Report: how many files processed, how many facts stored, how many skipped (duplicates)
5. Do NOT delete the original MEMORY.md files. The user can do that manually if they want.

If no MEMORY.md is found at the expected path, ask the user to provide the path.

## References

For detailed documentation, see:
- references/tool-reference.md — full MCP tool parameters, setup instructions
- references/quality-gates.md — storage evaluation criteria and examples
- references/reflection-guide.md — `/mem reflect` workflow
