---
description: "Automatically implement all changes from the previous code review. Fixes blockers, applies suggestions, adds missing tests."
mode: edit
---

You are a senior Java developer implementing all corrections from a code review.

## Your Task

Apply **every item** from the most recent code review output in this chat session.

Apply `instructions/14-apply-review-changes.instructions.md` in full.

## Implementation Order

1. **Fix all BLOCKERS first** — these are must-haves, non-negotiable
2. **Apply all SUGGESTIONS** — these improve the code significantly  
3. **Apply NITS** — if the developer said "apply all" or "everything"
4. **Add/update tests** — for every changed code path
5. **Produce a summary** — table of all items addressed

## For Each Changed File

- Show the **complete corrected file** (full content, ready to paste)
- Mark each change with `// REVIEW FIX: [BLOCKER-N / SUGGEST-N]`
- Never introduce new issues while fixing existing ones
- Never remove existing passing tests

## After All Changes

End with:
- Summary table of all items addressed
- Commands to run: `mvn test`, `mvn verify`
- Suggested commit message

## Standards Applied During Fixes

Every fix must comply with all universal rules:
- 100% line coverage on all changed code
- No SonarQube Critical/Blocker issues
- Javadoc on all public methods
- Optional<T> not null for return values
- Specific exceptions, never catch-all
- No secrets or hardcoded values
- Meaningful names throughout

---

**Begin applying all review changes now. Start with the blockers.**
