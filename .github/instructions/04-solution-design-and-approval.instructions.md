---
applyTo: "**/*.java"
---

# Solution Design and Approval Rules

## When Solution Design Is Required

A formal solution design and approval step is mandatory when:

- The change involves a new API endpoint or changes to an existing API contract.
- The change requires a new database table, column, or index.
- The change crosses two or more service or module boundaries.
- The change affects authentication, authorisation, or security.
- The estimated effort is more than 2 story points / half a day.
- The feature could have significant performance or scalability implications.
- The ticket is a new feature (Story) rather than a straightforward bug fix.

## Step 1 — Discover and Propose Multiple Solutions

Before choosing a solution, always generate **at least 2 viable options**.

For each option, document:

```
## Option N: <Short Name>

### Summary
One paragraph describing the approach.

### Technical Detail
- Key components involved
- Data model changes (if any)
- API contract changes (if any)
- External dependencies introduced

### Pros
- List advantages

### Cons
- List disadvantages, risks, and trade-offs

### Effort Estimate
- Development: X days
- Testing: X days
- Total: X days

### Risk Level
Low / Medium / High — with justification
```

## Step 2 — Present Options for Approval

- Share the options document with your tech lead, architect, and/or product owner **before coding**.
- Use a Jira comment, Confluence page, or a draft PR (with `[DRAFT]` in the title) to present the options.
- Do not begin implementation until you have explicit written approval of one option.
- If no response within the agreed SLA (typically 1 business day for in-sprint work), escalate.

## Step 3 — Implement the Approved Option

Once an option is approved:

- Document the approved option and the rationale for choosing it in the PR description.
- Implement exactly what was approved — if scope changes are discovered during implementation, stop and re-align with the approver before continuing.
- Apply all coding standards from `01-new-development.instructions.md`.
- If the approved option introduces a new pattern, document it in the team's architecture decision record (ADR).

## Solution Design Quality Standards

### API Design Checklist

- [ ] Endpoint follows REST conventions and versioning strategy
- [ ] Request/response DTOs defined with full validation annotations
- [ ] OpenAPI/Swagger annotation added
- [ ] Error responses documented for all failure scenarios
- [ ] Backward compatibility confirmed or breaking change communicated

### Data Model Checklist

- [ ] Entity relationship diagram (ERD) updated or created
- [ ] Column types and constraints defined
- [ ] Indexes identified for all query patterns
- [ ] Foreign key relationships correct
- [ ] Flyway/Liquibase migration script created and tested

### Performance Checklist

- [ ] Database query plan reviewed (`EXPLAIN ANALYZE`)
- [ ] Pagination implemented for list endpoints (never return unbounded lists)
- [ ] Caching strategy defined where applicable
- [ ] Async processing identified for long-running operations

### Security Checklist

- [ ] Authentication required? Confirmed.
- [ ] Authorisation rules defined per role
- [ ] Input validated server-side
- [ ] Sensitive data identified and protected
- [ ] No business logic leakage in error messages

## Architecture Decision Record (ADR) Template

When introducing a significant new pattern, create an ADR in `docs/adr/`:

```markdown
# ADR-XXXX: <Title>

## Status
Proposed / Accepted / Deprecated / Superseded by ADR-YYYY

## Context
What situation or problem caused this decision to be made?

## Decision
What was decided and why?

## Consequences
What are the positive and negative consequences of this decision?

## Alternatives Considered
What other options were evaluated and why were they rejected?

## Jira Reference
PROJ-XXXX
```
