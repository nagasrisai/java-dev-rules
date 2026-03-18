---
applyTo: "**/*.java"
---

# Java Bug Fixes and Enhancements Rules

## Before Touching Existing Code

- Read and fully understand the existing code before making any change.
- Run the existing test suite to confirm the baseline — know what passes and what fails before your change.
- Identify the root cause of the bug, not just the symptom. Do not patch symptoms.
- For enhancements, confirm the change does not break existing behaviour (backward compatibility).
- Check git history (`git log -p`) to understand why the code was written the way it was.

## Diagnosing a Bug

Follow this process in order:

1. **Reproduce the bug** — write a failing test that captures the exact broken behaviour first.
2. **Isolate the root cause** — trace through the call stack, inspecting inputs and outputs at each layer.
3. **Identify the blast radius** — determine all other code paths that call the same methods or share the same state.
4. **Fix the root cause** — change only what is necessary to fix the underlying issue.
5. **Verify the fix** — the previously failing test must now pass; the rest of the suite must remain green.

## Making the Fix

- Apply the **Minimal Change Principle** — change only what is necessary to fix the bug or deliver the enhancement.
- Do not refactor unrelated code in the same commit — keep the fix focused.
- If you discover technical debt during the fix, create a separate Jira ticket for it; do not fix it inline.
- When fixing a bug, add a comment referencing the Jira ticket: `// Fix for JIRA-1234: <brief description>`
- Update all relevant unit and integration tests.
- If the fix changes an API contract, document the change clearly and update Swagger/OpenAPI specs.

## Enhancement Rules

- Enhancements must be backward compatible unless explicitly agreed otherwise.
- Add feature flags / configuration toggles for major behavioural changes where possible.
- New logic added inside an existing method must be covered by new tests targeting that specific path.
- If a method grows beyond 40 lines due to an enhancement, extract the new logic into a private method.

## Regression Prevention

- Every bug fix MUST be accompanied by a new test that proves the bug is fixed and prevents regression.
- Run full test suite after fix: `mvn test` or `./gradlew test`.
- Run SonarQube/static analysis to confirm no new issues were introduced.
- If coverage drops below 100% for the changed class, add missing tests before committing.

## Code Quality During Fixes

- Do not introduce new code smells while fixing bugs:
  - No magic numbers or strings — use named constants
  - No deeply nested conditionals — extract into well-named methods
  - No duplicate code — extract to shared utility if the same logic appears twice
- Fix any pre-existing SonarQube issues in lines you are already modifying (opportunistic cleanup is acceptable, but keep it in a separate commit).

## Commit Message Format

Use conventional commits format:

```
fix(scope): short description of what was fixed

- Root cause: <description>
- Jira: JIRA-XXXX
- Breaking change: no / yes — <detail if yes>
```

For enhancements:

```
feat(scope): short description of the enhancement

- Jira: JIRA-XXXX
- Backward compatible: yes / no
```

## Checklist Before Raising Pull Request

- [ ] Failing test written first (bug fix) or acceptance tests written (enhancement)
- [ ] Root cause fixed, not just the symptom
- [ ] No unrelated changes in the commit
- [ ] All tests passing (`mvn test` or `./gradlew test`)
- [ ] 100% line coverage maintained for changed classes
- [ ] SonarQube scan clean — no new critical or blocker issues
- [ ] Jira ticket number referenced in commit message and PR description
- [ ] PR description explains: what the bug was, what the root cause was, and how it was fixed
- [ ] Reviewers tagged
