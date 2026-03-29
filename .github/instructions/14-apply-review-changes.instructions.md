---
applyTo: "**/*.java,**/*.xml,**/*.gradle,**/*.yaml,**/*.yml"
---

# Apply Review Changes — Automatic Implementation Rule

## When This Rule Activates

This rule activates when the developer says:
- "Apply all the changes you suggested"
- "Fix all the issues you found"
- "Implement the review feedback"
- "Make all those corrections"
- "Go ahead and fix everything"
- "Apply your suggestions"
- "Fix the blockers"
- "Now implement all of that"

## What You Must Do

Take the prior code review output (from rule 13 or rule 09) and implement **every single item** — blockers first, then suggestions, then nits if the developer asked for all.

Do NOT ask permission for each individual change. The developer has already approved the review by saying "apply". Act on all of it.

---

## Implementation Process — Follow This Exactly

### Phase 1: Triage the Review Items

Read the prior review output and categorise every item:

```
BLOCKERS:  [list each BLOCKER-N with file + description]
SUGGESTS:  [list each SUGGEST-N with file + description]
NITS:      [list each nit with file + description — apply only if developer said "apply all"]
```

### Phase 2: Apply Changes File by File

For each file that has changes:

1. Show the **complete corrected file** (not just the changed lines — show the full file so it can be pasted directly)
2. Label each change inline with `// REVIEW FIX: [BLOCKER-N / SUGGEST-N]` comments
3. Explain what was changed and why in 1-2 sentences above the code block

### Phase 3: Update or Add Tests

After fixing every code file:
- If a blocker involved missing test coverage → write the missing tests
- If a bug was fixed → write a regression test that proves it
- If a new method was added during fix → write unit tests for it
- Show complete test file content

### Phase 4: Verification Summary

After all changes are applied, produce this summary:

```markdown
## Changes Applied Summary

### Files Modified
| File | Changes Applied | Reason |
|---|---|---|
| OrderService.java | BLOCKER-1, SUGGEST-2 | Fixed null check + extracted method |
| OrderServiceTest.java | New tests | Coverage for null path + refactored method |

### Review Items Addressed
| Item | Status | Notes |
|---|---|---|
| BLOCKER-1: Null check missing | ✅ Fixed | Added Objects.requireNonNull with message |
| BLOCKER-2: Exception swallowed | ✅ Fixed | Rethrowing as OrderProcessingException |
| SUGGEST-1: Method too long | ✅ Applied | Extracted validateOrder() private method |
| SUGGEST-2: Magic number | ✅ Applied | Replaced 30 with MAX_RETRY_COUNT constant |
| [nit] Missing Javadoc | ✅ Applied | Added Javadoc to all public methods |

### Post-Fix Checklist
- [ ] All blockers resolved
- [ ] All suggestions applied
- [ ] Tests added for every fixed path
- [ ] 100% line coverage on all changed classes
- [ ] SonarQube self-check: no new Critical/Blocker issues
- [ ] Javadoc present on all public methods
- [ ] No hardcoded values remaining
- [ ] Ready to commit
```

---

## How to Apply Changes — Code Standards

When fixing code, apply ALL of these at the same time. Do not fix one thing and re-introduce another problem.

### Fixing a Missing Null Check

```java
// BEFORE (from review — BLOCKER-1: NPE risk)
public OrderDto processOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).get();
    return mapper.toDto(order);
}

// AFTER (REVIEW FIX: BLOCKER-1)
/**
 * Processes the order with the given ID and returns its DTO representation.
 *
 * @param orderId the ID of the order to process; must not be null
 * @return the processed order as a DTO
 * @throws ResourceNotFoundException if no order exists with the given ID
 */
public OrderDto processOrder(Long orderId) {
    Objects.requireNonNull(orderId, "orderId must not be null");
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + orderId));
    return mapper.toDto(order);
}
```

### Fixing a Swallowed Exception

```java
// BEFORE (from review — BLOCKER-2: exception swallowed)
try {
    paymentGateway.charge(amount);
} catch (Exception e) {
    log.error("Payment failed");
}

// AFTER (REVIEW FIX: BLOCKER-2)
try {
    paymentGateway.charge(amount);
} catch (PaymentGatewayException e) {
    log.error("Payment gateway charge failed for amount={}: {}", amount, e.getMessage(), e);
    throw new PaymentProcessingException("Payment failed: " + e.getMessage(), e);
}
```

### Fixing a Method That Is Too Long

```java
// BEFORE (from review — SUGGEST-1: method is 65 lines, does 3 things)
public Order placeOrder(PlaceOrderRequest request) {
    // 20 lines of validation
    // 25 lines of inventory check and reservation
    // 20 lines of order creation and persistence
}

// AFTER (REVIEW FIX: SUGGEST-1)
/**
 * Places a new order based on the given request.
 */
public Order placeOrder(PlaceOrderRequest request) {
    validateOrder(request);
    reserveInventory(request.getItems());
    return createAndPersistOrder(request);
}

private void validateOrder(PlaceOrderRequest request) {
    // validation logic — focused, under 20 lines
}

private void reserveInventory(List<OrderItemRequest> items) {
    // inventory logic — focused, under 20 lines
}

private Order createAndPersistOrder(PlaceOrderRequest request) {
    // order creation logic — focused, under 20 lines
}
```

### Fixing Missing Tests (After Code Fix)

```java
// New test for the fixed null check (BLOCKER-1 regression test)
@Test
@DisplayName("Should throw NullPointerException when orderId is null")
void shouldThrowWhenOrderIdIsNull() {
    assertThatThrownBy(() -> orderService.processOrder(null))
        .isInstanceOf(NullPointerException.class)
        .hasMessageContaining("orderId must not be null");
}

// New test for the fixed empty Optional (BLOCKER-1 path 2)
@Test
@DisplayName("Should throw ResourceNotFoundException when order not found")
void shouldThrowResourceNotFoundWhenOrderDoesNotExist() {
    when(orderRepository.findById(999L)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> orderService.processOrder(999L))
        .isInstanceOf(ResourceNotFoundException.class)
        .hasMessageContaining("Order not found: 999");
}
```

---

## What NOT to Do When Applying Changes

- Do NOT skip any blocker — all blockers must be fixed
- Do NOT introduce new issues while fixing — self-check against all universal rules
- Do NOT change things that were not in the review — minimal change principle within each fix
- Do NOT remove tests while refactoring — if you restructure code, keep all existing tests passing
- Do NOT apply a suggestion in a way that creates a new blocker
- Do NOT ask "should I fix this one?" for each item — fix them all, then present the summary

---

## After Applying All Changes

Always end with:

```
All [N] review items applied. Here is what to do next:

1. Run: mvn test (or ./gradlew test) — all tests must pass
2. Run: mvn verify — check JaCoCo coverage report
3. Run SonarQube scan locally if available
4. Review the diff one more time with: git diff
5. Commit with message referencing the review:
   fix: apply code review corrections [PROJ-XXXX if applicable]
```
