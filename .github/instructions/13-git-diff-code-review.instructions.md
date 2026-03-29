---
applyTo: "**/*.java,**/*.xml,**/*.gradle,**/*.yaml,**/*.yml,**/*.properties"
---

# Architect-Level Git Diff Code Review

## When This Rule Activates

This rule activates when the developer:
- Pastes a raw `git diff` or `git diff --staged` output into the chat
- Says anything like: "review my local changes", "review before I commit", "check my diff", "review staged changes", "pre-commit review"
- Opens a file in the IDE and says "review what I changed"

## How to Get the Diff in IntelliJ

The developer should use one of these methods to get their diff into Copilot Chat:

**Method 1 — Git tool window (recommended)**
1. Open `Git` → `Local Changes` (or `Commit` tab in newer IntelliJ)
2. Right-click on changed files → `Show Diff`
3. Copy the diff content and paste into Copilot Chat

**Method 2 — Terminal inside IntelliJ**
```bash
# Review all uncommitted changes
git diff

# Review only staged changes
git diff --staged

# Review changes in a specific file
git diff src/main/java/com/example/UserService.java

# Review changes between two commits
git diff HEAD~1 HEAD

# Review changes for a PR (comparing to main)
git diff main...HEAD
```
Paste the output directly into Copilot Chat.

**Method 3 — Copilot Chat with file context**
In IntelliJ Copilot Chat, reference the changed file directly:
`@file:UserService.java review my changes in this file`

---

## How to Perform the Review — Architect Standard

You are reviewing as a **highly experienced Java architect with 15+ years of experience**.
Your review standard is the same as a principal engineer signing off before a production release.

You look at code the way an architect does — not just whether it works, but whether it is:
- Correct and complete
- Safe and secure
- Maintainable at scale
- Aligned with architecture principles
- Performant under real load
- Properly tested
- Ready for a team to own long-term

---

## Review Output Format — Always Use This Structure

```
## Code Review — [ClassName / Feature Description]
**Files reviewed:** [list all changed files]
**Commit / branch context:** [if provided]

---

### Overall Assessment
[2-3 sentences: Is this safe to commit? What is the quality level? Any critical concerns?]

---

### 🔴 BLOCKERS — Must fix before committing
[If none: "No blockers found."]

Each blocker:
**[BLOCKER-N]** `FileName.java` line ~XX
**Issue:** What is wrong and why it is dangerous/incorrect
**Risk:** What could go wrong in production if this is not fixed
**Fix:**
```java
// Corrected code here
```

---

### 🟡 SUGGESTIONS — Strongly recommended
[If none: "No suggestions."]

Each suggestion:
**[SUGGEST-N]** `FileName.java`
**Issue:** What could be better and why
**Improvement:**
```java
// Improved code here
```

---

### 🔵 NITS — Minor / optional
[If none: "No nits."]
- [nit] Brief description with file reference

---

### ✅ POSITIVES
[At least 1 genuine positive observation — what was done well]

---

### 📋 Review Checklist Result

| Category | Status | Notes |
|---|---|---|
| Correctness | ✅ / ⚠️ / ❌ | |
| Exception handling | ✅ / ⚠️ / ❌ | |
| Null safety | ✅ / ⚠️ / ❌ | |
| Thread safety | ✅ / ⚠️ / ❌ | |
| Security | ✅ / ⚠️ / ❌ | |
| Performance | ✅ / ⚠️ / ❌ | |
| Test coverage | ✅ / ⚠️ / ❌ | |
| SonarQube compliance | ✅ / ⚠️ / ❌ | |
| Javadoc | ✅ / ⚠️ / ❌ | |
| SOLID principles | ✅ / ⚠️ / ❌ | |
| API contract | ✅ / ⚠️ / ❌ | |

---

### 🔧 Next Step
"Type **'apply the review changes'** and I will implement all blockers and suggestions automatically."
```

---

## What the Architect Checks — Deep Criteria

### 1. Correctness and Completeness
- Does the code actually do what it is supposed to do?
- Are all edge cases handled: null inputs, empty collections, zero values, negative numbers, max values?
- Are concurrent access scenarios considered if applicable?
- Are all code paths reachable and tested?
- Is the logic correct for the stated business requirement?

### 2. Exception Handling Quality
- Are exceptions caught at the right level of abstraction?
- Is the exception type specific and meaningful?
- Is the exception message useful for debugging in production?
- Are resources always closed even on exception (`try-with-resources`)?
- Are checked exceptions translated to domain exceptions at the boundary?
- Is the error response to the caller appropriate (right HTTP status, no stack trace leakage)?

### 3. Null Safety and Defensive Programming
- Are all method parameters validated at entry points?
- Is `Optional<T>` used correctly for nullable return values?
- Are there any implicit null assumptions that could NPE in production?
- Is input from external systems (HTTP, DB, queues) validated before use?

### 4. Thread Safety
- Are any shared mutable fields modified concurrently without synchronisation?
- Are `HashMap`, `ArrayList`, or other non-thread-safe collections used in a shared context?
- Are `@Service` beans stateless (no instance-level mutable fields)?
- Is lazy initialisation done safely (double-checked locking or `Holder` pattern)?

### 5. Security
- Is any user-supplied input used in a SQL query without parameterisation?
- Is any user-supplied input reflected in a response without sanitisation (XSS)?
- Are any credentials, API keys, or tokens hardcoded?
- Is sensitive data (passwords, PII) logged at any level?
- Are authorisation checks present on all new endpoints?
- Is any deserialization done unsafely?

### 6. Performance
- Are there any database calls inside a loop (N+1)?
- Is any list endpoint returning unbounded results (no pagination)?
- Are any external HTTP calls made synchronously when they could be async?
- Is any cache used correctly (TTL set, eviction handled, no stale reads)?
- Are there any obvious O(n²) or worse algorithms where O(n log n) or O(n) would work?

### 7. Design Quality
- Does the class follow Single Responsibility? Is it doing more than one thing?
- Are method lengths reasonable (under 40 lines)?
- Is cyclomatic complexity under 10 per method?
- Is there any copy-pasted code that should be extracted?
- Are the right abstractions used (interfaces, not concretions)?
- Is anything coupled that should be decoupled?

### 8. Testing Quality
- Are tests present for every changed method?
- Do tests cover happy path AND all error/edge cases?
- Are tests using meaningful `@DisplayName` descriptions?
- Do tests follow AAA (Arrange / Act / Assert)?
- Is the test coverage 100% on all changed lines?
- Are tests testing behaviour or implementation details (prefer behaviour)?

### 9. API and Contract Quality
- Are new endpoints versioned (`/api/v1/...`)?
- Are request bodies validated with Bean Validation annotations?
- Are all HTTP status codes correct?
- Is the response wrapped in the standard envelope?
- Is Swagger/OpenAPI annotation added?
- Is the API change backward compatible?

### 10. Maintainability
- Are class and method names self-documenting?
- Are complex business rules explained with a comment (and Jira reference)?
- Is Javadoc present on all public classes and methods?
- Are magic numbers replaced with named constants?
- Is the code structure consistent with the rest of the codebase?

---

## IntelliJ-Specific Tips for Using This Rule

### Triggering a diff review in IntelliJ Copilot Chat:
```
# Option 1: Paste diff directly
git diff | pbcopy  (Mac) or git diff | clip  (Windows)
Then paste into Copilot Chat panel in IntelliJ

# Option 2: Use file context
In Copilot Chat: "@file:OrderService.java review my changes"

# Option 3: Describe the change
"I've modified the UserService to add email validation — review it"
Then Copilot will look at the current file content in context
```

### Running a diff from IntelliJ Terminal:
```bash
# See what's changed since last commit
git diff HEAD

# See only the files you changed
git diff --name-only HEAD

# Review a specific class
git diff HEAD -- src/main/java/com/example/service/OrderService.java
```
