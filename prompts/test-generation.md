# Prompt: Test Generation from Acceptance Criteria

## When to Use

You have structured acceptance criteria (Gherkin, bullet list, or table) and need a first draft of test cases. This works best for isolated features or endpoints — not end-to-end flows spanning multiple services.

## Input Template

Copy the section below into your AI tool. Replace `{{ ... }}` with your context.

---

**Context**
- Feature: {{ feature name }}
- Component/Module: {{ file path or module name }}
- Language/Framework: {{ e.g. Python/pytest, TypeScript/vitest, Go/stdlib }}
- Existing test patterns: {{ link to or excerpt of an existing test in the same area }}

**Acceptance Criteria**

```
{{ paste acceptance criteria here — Gherkin or bullet list preferred }}
```

**Task**

Generate a test file that covers:

1. **Happy path** — the primary success case
2. **Edge cases** — empty inputs, boundary values, type mismatches
3. **Error states** — explicit failure modes mentioned in criteria
4. **State transitions** — if the feature changes system state, test before/after

**Output Format**

Return a single code block with the full test file. Use the same framework and conventions as the `Existing test patterns` section. Include imports, fixtures/mocks, and assertions.

Do NOT:
- Add setup/teardown that doesn't exist in the existing patterns
- Generate mocks for dependencies that aren't mocked elsewhere in this project
- Add TODO comments or placeholder assertions

---

## Example Run

**Input criteria:**

```
Feature: Cart total calculation
  Scenario: Apply percentage discount
    Given a cart with items totaling $100
    When a 10% discount code is applied
    Then the total should be $90
```

**Output** (abbreviated):

```python
def test_apply_percentage_discount_reduces_total():
    cart = Cart([Item("A", 100.0)])
    cart.apply_discount(Discount.percentage(10))
    assert cart.total == 90.0

def test_apply_percentage_discount_with_empty_cart_returns_zero():
    cart = Cart([])
    cart.apply_discount(Discount.percentage(10))
    assert cart.total == 0.0

def test_apply_percentage_discount_with_negative_percentage_raises():
    cart = Cart([Item("A", 100.0)])
    with pytest.raises(InvalidDiscountError):
        cart.apply_discount(Discount.percentage(-5))
```

## Known Failure Modes

| Failure | Likely Cause |
|---------|-------------|
| Tests pass but miss the actual behavior boundary | AC was too vague — add concrete values |
| Generated mocks don't match real interfaces | Didn't provide enough context on existing patterns |
| Assertions are tautological (assert result == result) | Output format constraint wasn't clear; add a specific example |
