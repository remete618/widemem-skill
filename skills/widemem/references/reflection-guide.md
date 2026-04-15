# Memory Reflection — /mem reflect

A structured audit of all stored memories. Finds problems and proposes fixes. Always reports before making changes.

## Workflow

### Step 1: Gather all memories
- Call `widemem_export` to get the full memory set as JSON
- Parse the exported JSON to get all memories with their IDs, content, importance, and timestamps (created_at, updated_at)
- If export fails or returns empty, fall back to broad `widemem_search` queries:
  - "personal information" (top_k=50)
  - "preferences and likes" (top_k=50)
  - "work and projects" (top_k=50)
  - "relationships and people" (top_k=50)
  - "dates and events" (top_k=50)
  - Deduplicate results by memory ID

### Step 2: Analyze for issues
Scan collected memories for:

**Duplicates** — Two or more memories expressing the same fact.
Example: "Lives in San Francisco" and "User's city is San Francisco"

**Contradictions** — Memories that conflict with each other.
Example: "Works at Google" and "Works at Meta" (unless one is dated/outdated)

**Stale entries** — Memories referencing past states that may no longer be true. Use timestamps and content to detect:
- **Likely stale (flag):** Contains temporal language ("currently", "right now", "this week", "working on", "debugging") AND older than 7 days
- **Worth confirming (flag):** Any memory older than 90 days, regardless of content
- Example: "Currently debugging the auth module" (created_at: 3 weeks ago) = likely stale

**Consolidation candidates** — Multiple small facts that could merge.
Example: "Likes Python" + "Prefers typed languages" + "Uses mypy" could become "Python developer who prefers typed code with mypy"

### Step 3: Report findings
Present a structured report:

```
Memory Reflection Report
========================
Total memories: N

Duplicates (X found):
  - "fact A" (id: ...) == "fact B" (id: ...) -> keep A, delete B

Contradictions (Y found):
  - "fact A" vs "fact B" -> which is current?

Stale candidates (Z found):
  - "fact" (created: date) -> still relevant?

Consolidation groups (W found):
  - Group: [fact A, fact B, fact C] -> merge into "..."
```

### Step 4: Confirm before acting
Ask the user to approve each proposed change. Never delete or modify without confirmation.

### Step 5: Execute approved changes
- Delete duplicates via `widemem_delete`
- For contradictions: delete the wrong one, keep or update the correct one
- For stale: delete if confirmed outdated
- For consolidation: delete originals, store merged fact via `widemem_add`
