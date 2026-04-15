# Quality Gates — Storage Evaluation

Before storing any memory, evaluate whether it's worth persisting. This prevents memory bloat and keeps retrieval quality high.

## Classification Criteria

### SKIP — Do not store
- Greetings: "hi", "thanks", "ok", "got it"
- Tool output: command results, error messages, stack traces
- Ephemeral: "run the tests", "open that file", "try again"
- Meta-conversation: "that worked", "never mind", "let me think"

### LOW — Store only if explicitly requested
- Trivial preferences: "I like dark mode"
- One-time context: "I'm debugging this function right now"
- Temporary state: "I'm on the dev branch"

### MEDIUM — Store after dedup check
- Stated preferences: "I prefer TypeScript over JavaScript"
- Personal facts: "I work at Google", "I have two dogs"
- Project decisions: "We chose PostgreSQL for the database"
- Relationships: "Alice is my manager"

### HIGH — Store immediately
- Corrections: "No, my name is spelled with a K" (delete old + store new)
- Repeated info: user is telling you something again (they expect you to know)
- Important dates: birthdays, deadlines, milestones

### CRITICAL — Pin with high importance
- Explicit request: "Remember this", "Don't forget", "Save this"
- Medical/financial/legal facts (YMYL)
- Information the user was frustrated you didn't know

## Deduplication Procedure

Before storing MEDIUM or higher:
1. Search widemem with the core fact as query (top_k=3)
2. If a result has very high similarity and same meaning — skip (already stored)
3. If a result contradicts the new fact — check timestamps:
   - The newer fact wins by default (user's most recent statement is the current truth)
   - Delete the old memory, store the new one
   - If both facts could be simultaneously true (e.g., user works at two places), ask before deleting
4. If no relevant results — proceed with storage

## Timestamp-Aware Conflict Resolution

When a contradiction is found between an existing memory and new information:
- The `created_at` field on each memory indicates when it was stored
- Newer information from the user supersedes older stored facts
- When deleting the outdated memory, log which memory was replaced and why
- If the user corrects a fact ("Actually, I moved to Berlin"), treat the correction as HIGH priority regardless of the old memory's importance score

## Examples

| User says | Classification | Action |
|-----------|---------------|--------|
| "Thanks, that fixed it" | SKIP | Don't store |
| "I'm working on the auth module" | LOW | Skip unless asked |
| "I'm a data scientist at Stripe" | MEDIUM | Dedup check, then store |
| "Actually I moved to Berlin last month" | HIGH | Search for old location, delete, store new |
| "Remember: my API key rotates every 90 days" | CRITICAL | Pin via widemem_pin |
