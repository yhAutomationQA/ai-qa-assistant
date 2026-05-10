# AI Governance for QA Engineering

## The Problem This Addresses

AI-generated test automation looks plausible. That's the danger. A test that compiles, passes on green code, and follows syntactically correct patterns can still be **completely worthless** — or worse, create false confidence that a feature is tested when it isn't.

This document defines when to trust AI output, how to validate it, and when to reject it outright.

## When NOT to Trust AI-Generated Tests

These are hard no-nos. If the test falls into any of these categories, don't use AI output without fully rewriting it.

### 1. Security or authorization boundaries

AI does not understand your auth model. It will generate tests that use valid tokens without testing token validation, permission escalation, or role boundaries. If the test verifies that User A cannot access User B's data, write it yourself. AI will get the happy path right and the rejection wrong.

- **AI will**: test that an admin can access the admin panel
- **AI will NOT**: test that a deactivated admin's session is rejected, or that a JWT with manipulated claims is caught

### 2. Financial calculations involving rounding or precision

AI treats `0.1 + 0.2` as `0.3` in its "thinking" but doesn't know how your system handles floating-point rounding, tax tiers, or currency conversion. If the test verifies monetary amounts, you must hand-write the expected values.

- **AI will**: assert `total === 19.99` for a $19.99 item with 10% discount
- **AI will NOT**: account for your system's two-decimal ceiling rounding vs. bankers' rounding

### 3. Regulatory or compliance-required tests

GDPR deletion, PCI compliance, audit log completeness — these tests verify legal obligations, not correctness. AI doesn't know your compliance requirements. Do not delegate.

### 4. Tests for code that doesn't exist yet

AI is optimistic. Give it a spec for a future feature and it will generate tests that pass against imaginary code. These tests will either:
- Fail to compile because the interfaces are wrong
- Compile but be structurally coupled to assumptions that don't survive implementation
- Pass for the wrong reasons because the implementation differs from what the AI expected

Write tests after or during implementation, not before.

### 5. Complex state machines or multi-step workflows

If the feature involves state transitions (order: placed → confirmed → shipped → delivered), AI will miss intermediate states, fail to test invalid transitions, and over-test obvious paths.

- **AI will**: test that placed → confirmed works
- **AI will NOT**: test that shipped → placed is rejected, or that a delivery notification without confirmation leaves the order in a valid state

## Validation Requirements

Every AI-generated test must pass these checks before it enters the codebase.

### Mandatory: "Does it actually test something?"

The most common failure of AI-generated tests is that they execute but assert nothing meaningful. The test calls a function and checks that it doesn't crash, but doesn't verify the result.

**Reject if:**
- The assertion is tautological (`expect(result).toBe(result)`)
- The assertion only checks that no exception was thrown
- The test mocks everything and then asserts on the mock (testing the mock, not the code)
- The test uses `expect(true).toBe(true)` or equivalent

### Mandatory: "Does it fail when the code breaks?"

Take the test. Break the implementation in the way the test is supposed to catch. Does the test actually fail?

If the answer is "no", the test is noise. Delete it.

### Mandatory: "Does the test match project conventions?"

Run it past:
- Linter (same rules as the rest of the project)
- Type checker (strict mode)
- Test runner (does it actually pass in CI?)

If the AI used a pattern that doesn't exist anywhere else in the test suite (e.g., a mocking library no one else uses, a custom assertion helper it invented), reject it. Consistency matters more than cleverness.

### Recommended: "Can another engineer understand it in 30 seconds?"

AI tests tend to be verbose. They generate long descriptive names, deeply nested describes, and excessive comments. If a teammate would need to study the test to understand what it covers, rewrite it.

## Risks of Hallucinated QA Coverage

"Coverage" from AI-generated tests is not real coverage. The CI will show green, and your coverage tool will show higher numbers. Both are misleading.

### The "Coverage Without Testing" Trap

```typescript
// AI-generated
describe('processPayment', () => {
  it('calls payment gateway', async () => {
    const gateway = { charge: jest.fn().mockResolvedValue({ status: 'success' }) };
    const result = await processPayment(100, gateway);
    expect(gateway.charge).toHaveBeenCalled();
  });
});
```

This test achieves 100% line coverage of `processPayment`. It does not test:
- That the result is returned to the caller
- That failure is handled
- That the amount is passed correctly
- That the gateway is called with the right arguments (only that it's called at all)

The coverage number is real. The coverage is not.

### The "Green Test, Missed Bug" Scenario

An AI generates a test for a discount calculation. The test uses the same logic as the implementation to derive the expected value:

```typescript
// AI-generated
it('applies 10% discount', () => {
  const result = applyDiscount(100, 0.1);
  expect(result).toBe(100 - (100 * 0.1)); // tautological
});
```

The test passes. Two weeks later, someone changes the discount formula to compound. The test still passes because it computes the expected value the same way the implementation does. The bug ships.

**The fix:** AI-generated tests should use hardcoded expected values, not computed ones. If the spec says "10% off $100 = $90", the test should assert `90`, not `100 * 0.9`.

### The "Mock That Never Matches Reality" Problem

AI generates mocks based on your type definitions but doesn't know your runtime behavior. It will:
- Mock a method that doesn't exist in production (but exists in the type because it's optional)
- Return a value in a format the real service never uses
- Skip error paths that the real service returns regularly

**Validation:** Run the test against the real integration at least once. If the mock doesn't match reality, rewrite it.

## The Human Review

AI-generated tests need a different review approach than human-written ones. Human-written tests usually have intent but may contain bugs. AI-generated tests are usually syntactically flawless and semantically empty.

### Review Checklist

For every AI-generated test:

1. **Read the assertions first.** If the assertions don't make sense, the rest doesn't matter.
2. **Check what's mocked.** If the mock is unrealistic, the test is unreliable.
3. **Check what's NOT tested.** Look at the code the test claims to cover. What paths are missing?
4. **Run the "break the code" test.** Manually break the implementation and verify the test catches it.
5. **Remove noise.** Cut setup code that isn't used, assertions that duplicate other tests, and comments that explain what the test does (the test name should do that).

### When to Reject Instead of Edit

- **More than 30% of the test is wrong**: reject and regenerate with a more specific prompt, or write it yourself.
- **The test passes but couldn't catch the bug it's supposed to catch**: reject. It's worse than no test because it creates false confidence.
- **The test is correct but uses patterns that don't exist in the project**: reject. It will confuse future maintainers.
- **The test is correct but takes >5 seconds to run for a unit test**: reject and simplify. AI does not optimize for speed.

## Maintaining Test Quality Standards

### Consistency

Every AI-generated test should be indistinguishable from a hand-written test in the same project. If you can tell it was AI-generated (weird import order, overly descriptive names, excessive comments), it doesn't belong in the codebase.

**Rule:** If a reviewer can reliably identify which tests in your suite were AI-generated, your validation process is broken.

### Readability

AI favors verbosity over clarity. Common patterns to fix:

| AI Default | Better |
|-----------|--------|
| `const expectedResult = calculateExpectedValue(input)` | Hardcode the expected value |
| Multiple `it()` blocks that differ only by one parameter | Parameterized test |
| Comments explaining every line | Remove comments; make the code self-documenting |
| Deeply nested `describe` blocks (3+ levels) | Flatten or split into separate test files |

### Performance

AI doesn't consider test suite performance. It will happily generate:
- Tests that set up the same fixture 10 different ways instead of reusing it
- E2E tests for things that should be unit tests
- Tests that make real network calls when they should mock
- Tests that spin up a full application server to test a single function

**Review with a performance lens:** Will this test still be fast when there are 5000 of them?

### Maintainability

AI writes for the present. It doesn't know that `processPayment` will be renamed to `chargeCustomer` next month, or that the current mock library will be deprecated. The test should be resilient to refactoring.

- Prefer testing public interfaces, not internal implementation details
- Don't mock what you don't own (third-party libraries, language built-ins)
- Don't assert on implementation-specific calls unless the behavior is part of the contract

## Summary

| Situation | Action |
|-----------|--------|
| Security/auth test | Write manually |
| Financial calculation | Write expected values manually |
| Compliance test | Write manually |
| Test for code not yet written | Wait until code exists |
| AI test passes but doesn't fail when code breaks | Reject |
| AI test uses nonexistent patterns | Reject |
| AI test mocks unrealistically | Rewrite mock |
| AI test is correct but verbose | Edit for style |
| AI test is correct and clean | Accept after standard PR review |

## One-Sentence Philosophy

> The goal is not to maximize the number of AI-generated tests in your suite. The goal is to have zero tests in your suite that you can't trust, regardless of who or what wrote them.
