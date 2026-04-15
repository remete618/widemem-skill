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

### MEDIUM — Store via widemem_add (SDK handles dedup)
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

## Deduplication and Conflict Resolution

The SDK handles deduplication and conflict resolution internally when you call `widemem_add`. You do NOT need to search before storing. The SDK pipeline:
1. Extracts facts from text via LLM
2. Searches the vector store for existing similar memories
3. Runs batch conflict resolution (LLM-based) to decide: ADD, UPDATE, or SKIP
4. Content-hash checks prevent exact duplicates

If the SDK detects ambiguous conflicts, it returns a `clarifications` field. Surface these to the user.

## Timestamp-Aware Conflict Resolution

When a contradiction surfaces (during reflection, or via SDK clarifications):
- The `created_at` field on each memory indicates when it was stored
- Newer information from the user supersedes older stored facts
- When deleting the outdated memory, log which memory was replaced and why
- If the user corrects a fact ("Actually, I moved to Berlin"), treat the correction as HIGH priority regardless of the old memory's importance score

## Examples

| User says | Classification | Action |
|-----------|---------------|--------|
| "Thanks, that fixed it" | SKIP | Don't store |
| "I'm working on the auth module" | LOW | Skip unless asked |
| "I'm a data scientist at Stripe" | MEDIUM | Store via `widemem_add` (SDK dedup) |
| "Actually I moved to Berlin last month" | HIGH | Store via `widemem_add` (SDK resolves conflict) |
| "Remember: my API key rotates every 90 days" | CRITICAL | Pin via widemem_pin |
