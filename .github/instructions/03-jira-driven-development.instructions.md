---
applyTo: "**/*.java"
---

# Jira-Driven Development Rules

## Picking Up a Jira Ticket

Before writing a single line of code, complete the following steps in order:

### Step 1 — Read and Understand the Ticket

- Read the **entire** Jira ticket: title, description, acceptance criteria, and all comments.
- Identify the ticket type:
  - **Story** — new user-facing feature
  - **Task** — technical work with no direct user story
  - **Bug** — defect fix
  - **Spike** — research/investigation with a defined time-box
  - **Sub-task** — part of a larger story
- If the acceptance criteria are missing or ambiguous, do NOT start coding — ask the product owner or tech lead to clarify and update the ticket before you proceed.

### Step 2 — Understand the Requirement Fully

Ask yourself (and, if needed, the team):

1. **Who** is this for? What user role or system is affected?
2. **What** exactly needs to change or be created?
3. **Why** is this needed? What business value does it deliver?
4. **When** does it apply? Are there edge cases, time-based conditions, or environment-specific rules?
5. **How** does it interact with existing functionality? What are the dependencies?
6. **What are the non-functional requirements?** (Performance, security, scalability, backward compatibility)

### Step 3 — Clarify Before Building

If any of the following are missing from the ticket, get answers before starting:

- Acceptance criteria (specific, measurable, testable)
- Error handling expectations (what should happen on failure?)
- Performance expectations (e.g., "must respond within 200ms")
- Security requirements (authentication, authorisation, data sensitivity)
- API contract details (request/response structure)
- Data migration requirements (if schema changes are needed)

Document the answers as comments on the Jira ticket and/or in the PR description.

### Step 4 — Estimate and Break Down

- If the ticket is large, break it into sub-tasks on Jira before coding.
- Each sub-task should be completable in under 2 working days.
- Estimate in story points or hours — update the Jira ticket with your estimate.

## Branch Naming Convention

Always branch from the correct base branch (typically `develop` or `main` depending on your Git workflow):

```
<type>/<jira-number>-<short-description>

Examples:
feature/PROJ-1234-add-user-registration
bugfix/PROJ-5678-fix-null-pointer-in-payment
task/PROJ-9012-migrate-legacy-config
```

## Coding With Jira Context

- Reference the Jira ticket in every commit message:
  ```
  feat(auth): add JWT refresh token support

  - Implements PROJ-1234 acceptance criteria
  - Adds /api/v1/auth/refresh endpoint
  - 100% unit and integration test coverage
  ```
- Add the Jira ticket number as a comment in complex logic areas:
  ```java
  // PROJ-1234: Business rule — users with SUSPENDED status cannot refresh tokens
  if (user.getStatus() == UserStatus.SUSPENDED) {
      throw new AccountSuspendedException("Account is suspended");
  }
  ```

## Updating the Jira Ticket

Keep the Jira ticket current throughout development:

| Your action | Jira update |
|---|---|
| Starting work | Move to **In Progress**; assign to yourself |
| Blocked on a dependency | Add a comment; link the blocker ticket; move to **Blocked** |
| Implementation complete | Move to **In Review**; add PR link as a comment |
| PR approved and merged | Move to **Done** or **Ready for QA** per your workflow |

## Pull Request Description Template

Use this template when raising a PR for a Jira ticket:

```markdown
## Jira Ticket
[PROJ-XXXX](https://yourcompany.atlassian.net/browse/PROJ-XXXX)

## What Was Done
Brief description of what was implemented.

## Why
Explanation of the business requirement driving this change.

## How
Key technical decisions made during implementation.

## Testing
- Unit tests: [X new tests, all passing]
- Integration tests: [X new tests, all passing]
- Manual testing: [describe what you tested manually and results]

## Checklist
- [ ] Acceptance criteria met (verified against Jira ticket)
- [ ] Unit tests written — 100% line coverage
- [ ] Integration tests written
- [ ] SonarQube clean — no new issues
- [ ] API documentation updated (if applicable)
- [ ] Database migration added (if applicable)
- [ ] Jira ticket updated and PR link added
```
