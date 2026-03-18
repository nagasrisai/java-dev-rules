---
applyTo: "**/*.java"
---

# Java Code Review Rules

## Purpose of a Code Review

Code review is not about finding fault — it is about:

1. Catching bugs before they reach production.
2. Sharing knowledge across the team.
3. Enforcing agreed standards consistently.
4. Improving the long-term health of the codebase.

A review comment should help the author improve, not demonstrate the reviewer's knowledge.

---

## How to Give Review Feedback

### Tone and Language

- Be respectful and constructive — review the code, not the person.
- Use "we" language where possible: "We usually..." instead of "You should..."
- Explain WHY a change is needed, not just WHAT to change.
- If a comment is a personal preference rather than a rule, label it clearly: `[nit]` or `[optional]`.
- Praise good code openly — comments do not have to be negative.

### Comment Categories — Always Label Your Comments

| Label | Meaning |
|---|---|
| `[blocker]` | Must be fixed before merge — bug, security issue, or clear rule violation |
| `[suggestion]` | Strong recommendation but not a hard blocker — explain your reasoning |
| `[nit]` | Minor style or preference issue — author may address or not |
| `[question]` | Genuinely seeking to understand the intent — not a criticism |
| `[praise]` | Recognising good work |

Example review comments:

```
[blocker] This method catches Exception broadly and swallows it silently.
If the payment service throws, the caller will never know. 
Please catch specific exceptions and either handle or rethrow with context.

[suggestion] Consider extracting this block into a private method named 
`calculateAdjustedAmount()`. The current method is 52 lines and handles 
two distinct concerns, which makes it harder to test in isolation.

[nit] Minor: variable name `d` on line 34 could be more descriptive — 
`discountAmount` would make this self-documenting.

[question] Why is this check done after the database save rather than before? 
Is there a specific ordering requirement I'm missing?
```

---

## What to Check in Every Java PR Review

### Correctness

- [ ] Does the code do what the Jira ticket / acceptance criteria require?
- [ ] Are all edge cases handled (null inputs, empty collections, boundary values)?
- [ ] Are error conditions handled correctly with appropriate HTTP status codes?
- [ ] Does the logic handle concurrent access safely (if applicable)?
- [ ] Are all database operations transactional where required (`@Transactional`)?

### Code Quality

- [ ] Are class and method names clear and intention-revealing?
- [ ] Is cyclomatic complexity per method 10 or below?
- [ ] Are methods under 40 lines?
- [ ] Is there any duplicate code that should be extracted?
- [ ] Are there any magic numbers or strings that should be constants?
- [ ] Are all `TODO` comments linked to a Jira ticket?

### Testing

- [ ] Are unit tests present for every new method?
- [ ] Are integration tests present for new or changed API endpoints?
- [ ] Is line coverage 100% for the changed classes?
- [ ] Do tests follow AAA (Arrange / Act / Assert) structure?
- [ ] Do tests cover both happy path and all error/edge cases?
- [ ] Are test names descriptive (`@DisplayName` used)?

### Security

- [ ] Is all user input validated server-side?
- [ ] Are there any hardcoded credentials, tokens, or secrets?
- [ ] Are sensitive fields excluded from logs?
- [ ] Is the correct authorisation applied to new endpoints?
- [ ] Are SQL queries parameterised (no string concatenation)?

### Performance

- [ ] Are there any N+1 query problems?
- [ ] Does any new endpoint return an unbounded list without pagination?
- [ ] Are external service calls outside of loops?
- [ ] Are timeouts configured on all outbound HTTP calls?

### API Design

- [ ] Is the endpoint versioned (`/api/v1/...`)?
- [ ] Are request DTOs validated with Bean Validation annotations?
- [ ] Is the response wrapped in the standard `ApiResponse<T>` envelope?
- [ ] Is Swagger/OpenAPI annotation added?
- [ ] Are all possible error responses documented?

### SonarQube

- [ ] Does the CI SonarQube scan pass with zero new Critical or Blocker issues?
- [ ] Are all new Security Hotspots reviewed and resolved?

---

## Reviewing as a Copilot Assistant

When asked to review code, follow this structure in your response:

### 1. Summary
One paragraph stating your overall assessment. Is this safe to merge with changes? Does it need significant rework? Are there blockers?

### 2. Blockers (if any)
List any `[blocker]` issues with file and line number references. The PR cannot merge until these are resolved.

### 3. Suggestions
List `[suggestion]` items with reasoning. These are strongly recommended but the author may make a final call.

### 4. Nits
List `[nit]` items briefly. Keep this section short.

### 5. Positives
Note at least one thing the author did well.

### 6. Revised Code (if applicable)
If a blocker requires a non-trivial fix, provide the corrected code with an explanation.

---

## Author — Before Requesting a Review

Self-review your own PR against this checklist before tagging reviewers:

- [ ] I have read every line of my own diff.
- [ ] All CI checks pass (tests, coverage, SonarQube, style).
- [ ] I have resolved all TODO items or linked them to Jira tickets.
- [ ] PR description is complete (what, why, how, test scenarios table).
- [ ] Jira ticket is updated and PR link is added.
- [ ] I have removed all debug logs, commented-out code, and print statements.
- [ ] I have not added any dependency without checking it is necessary and approved.
