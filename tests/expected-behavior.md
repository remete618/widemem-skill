# Expected Behavior Test Cases

Manual validation scenarios for the widemem skill. Each scenario describes a user input, the expected classification, and the expected action. Use these to verify the skill behaves correctly after changes.

## Quality Gate Tests

### SKIP tier (should NOT store)

| # | User says | Expected | Why |
|---|-----------|----------|-----|
| 1 | "ok" | SKIP, no storage | Greeting/acknowledgment |
| 2 | "thanks, that fixed it" | SKIP, no storage | Meta-conversation |
| 3 | "run the tests" | SKIP, no storage | Ephemeral command |
| 4 | "never mind" | SKIP, no storage | Meta-conversation |
| 5 | "Error: ModuleNotFoundError..." | SKIP, no storage | Tool output |

### LOW tier (store only if explicitly asked)

| # | User says | Expected | Why |
|---|-----------|----------|-----|
| 6 | "I like dark mode" | LOW, skip unless asked | Trivial preference |
| 7 | "I'm on the dev branch right now" | LOW, skip unless asked | Temporary state |
| 8 | "I'm debugging the auth module" | LOW, skip unless asked | One-time context |

### MEDIUM tier (store via widemem_add, SDK handles dedup)

| # | User says | Expected | Why |
|---|-----------|----------|-----|
| 9 | "I work at Stripe" | MEDIUM, call widemem_add directly (no pre-search) | Personal fact |
| 10 | "I prefer TypeScript over JavaScript" | MEDIUM, call widemem_add directly | Stated preference |
| 11 | "We chose PostgreSQL for the database" | MEDIUM, call widemem_add directly | Project decision |
| 12 | "Alice is my manager" | MEDIUM, call widemem_add directly | Relationship |

### HIGH tier (store immediately)

| # | User says | Expected | Why |
|---|-----------|----------|-----|
| 13 | "No, my name is spelled with a K" | HIGH, call widemem_add (SDK resolves conflict) | Correction |
| 14 | User repeats "I live in Berlin" (second time) | HIGH, pin it | Repeated info |
| 15 | "My deadline is April 30th" | HIGH, store | Important date |

### CRITICAL tier (pin)

| # | User says | Expected | Why |
|---|-----------|----------|-----|
| 16 | "Remember this: my API key rotates every 90 days" | CRITICAL, pin at 9.0 | Explicit "remember" |
| 17 | "Don't forget, I'm allergic to penicillin" | CRITICAL, pin at 9.0 | YMYL + explicit request |

## Confidence Response Tests

| # | Scenario | Expected response style |
|---|----------|------------------------|
| 18 | Search returns confidence: HIGH | HIGH: answer confidently, "I recall you mentioned..." (use the confidence field, don't assess yourself) |
| 19 | Search returns similarity 0.55 | MODERATE: answer with caveat, "I think..." |
| 20 | Search returns similarity 0.30 | LOW: "I have a vague recollection, but I'm not sure..." |
| 21 | Search returns similarity 0.10 | NONE: "I don't have that stored. Want to tell me?" |
| 22 | Search returns no results | NONE: never fabricate, offer to save |

## Dedup and Conflict Tests (handled by SDK)

| # | Existing memory | User says | Expected |
|---|----------------|-----------|----------|
| 23 | "Works at Stripe" | "I work at Stripe" | SDK skips (near-duplicate via content hash) |
| 24 | "Works at Stripe" | "I just joined Google" | SDK resolves conflict (LLM-based batch resolution) |
| 25 | "Lives in San Francisco" | "I moved to Berlin last month" | SDK resolves conflict (update/replace) |
| 26 | "Works at Google" | "I also freelance for Acme" | SDK may return clarification; if so, surface to user |

## Frustration Detection Tests

| # | User says | Expected |
|---|-----------|----------|
| 27 | "I ALREADY told you my anniversary is March 15th" | Detect frustration, search harder, then pin when repeated |
| 28 | "You should know this by now" | Broader search, apologize if not found, pin when repeated |

## Import Tests

| # | Scenario | Expected |
|---|----------|----------|
| 29 | `/mem import` with valid MEMORY.md | Read index, follow links, extract facts, store via widemem_add, report count |
| 30 | `/mem import` with no MEMORY.md at expected path | Ask user for the correct path |
| 31 | `/mem import` when some facts already exist in widemem | Skip duplicates, report how many skipped |

## Reflection Tests

| # | Scenario | Expected |
|---|----------|----------|
| 32 | `/mem reflect` with <100 memories | Export all, analyze full set, report findings |
| 33 | Two memories: "Lives in SF" + "User's city is San Francisco" | Flag as duplicate, propose keeping one |
| 34 | Two memories: "Works at Google" + "Works at Meta" | Flag as contradiction, ask which is current |
| 35 | Memory: "Debugging auth module" (created_at: 3 weeks ago) | Flag as stale (temporal keyword "debugging" + older than 7 days) |
| 37 | Memory: "I work at Stripe" (created_at: 4 months ago) | Flag as worth confirming (older than 90 days, no temporal keyword) |
| 38 | Memory: "I'm on the dev branch" (created_at: 2 days ago) | Do NOT flag (temporal keyword but only 2 days old) |
| 36 | "Likes Python" + "Uses mypy" + "Prefers typed code" | Flag as consolidation candidate |
