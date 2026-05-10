# Workflow: Identifying Regression Risks During PR Review

## Scenario

A developer opens a PR. The diff is 400+ lines across 12 files. The title is "Refactor payment processing to use new provider SDK." No test changes are included. You need to assess regression risk before this merges.

## Step-by-Step

### 1. Understand the diff's shape

Before involving AI, categorize the change:

| Metric | Value | Signal |
|--------|-------|--------|
| Files changed | 12 | High — touches multiple modules |
| Core logic vs. config | Core (payment flow) | High — central domain |
| Test changes | 0 | Red flag — no tests updated for a refactor |
| New dependencies | 1 (provider SDK v2) | High — API surface change |
| Deleted code | ~150 lines | Could be clean or could be deleted error handling |
| Interface changes | Maybe — imports changed | Need to check callers |

### 2. Map the dependency graph with AI

Feed the PR description and diff summary to the AI:

```
PR: {{ title }}
Description: {{ description }}
Files changed: {{ list }}
Diff highlights (key hunks):
```diff
{{ paste the most significant hunks — the ones that change return types, error handling, or method signatures }}
```

Task:
1. Identify every public function/class whose signature changed
2. Trace all callers of those functions (internal and in test files)
3. Flag any error-handling paths that were removed or changed
4. Note what the old SDK provided that the new SDK doesn't (method removals, changed return types)

This is a code comprehension task, not a test generation task. Focus on blast radius.
```

**What you'll get back:**

```
## 1. Signature Changes
- `processPayment(amount: number, currency: string)` → `processPayment(amount: Money)` 
  - `Money` is a new type: `{ amount: number; currency: string; precision: number }`
  - Tests that call processPayment with two args will break

- `refundPayment(transactionId: string)` now returns `RefundResult` instead of `boolean`
  - `RefundResult` has `status: 'success' | 'pending' | 'failed'` and `message: string`
  - Old callers checking for `true`/`false` will misbehave

## 2. Affected Callers
| Caller | File | Existing Tests? |
|--------|------|----------------|
| checkoutService.createOrder → processPayment | src/orders/checkout.ts | test/orders/checkout.test.ts (3 tests) |
| adminPortal.manualRefund → refundPayment | src/admin/refund.ts | No tests exist |
| subscriptions.cancel → refundPayment | src/billing/subscriptions.ts | test/billing/subscriptions.test.ts (2 tests, none for refund path) |

## 3. Removed Error Handling
- Old SDK threw `PaymentGatewayTimeout` on timeout. New SDK returns `{ status: 'pending' }` instead.
  - Callers that catch `PaymentGatewayTimeout` will never trigger — the error is silently swallowed.
  - File: `src/payments/handler.ts` lines 45-52 — dead catch block that should now poll for status.

## 4. SDK Capability Changes
- Old SDK supported `voidPayment()`. New SDK does not. 
  - `adminPortal.voidTransaction` calls this. No test covers it. The call will fail at runtime.
```

### 3. Assess test coverage gaps

Now that you know the blast radius, cross-reference against the actual test suite. Use the `regression-impact-analysis.md` prompt with the full test map.

**Key findings from step 2:**

| Risk | Coverage | Action |
|------|----------|--------|
| `processPayment` signature change | checkout.test.ts has 3 tests — need to verify they use the old signature | Must update existing tests |
| `refundPayment` return type change | No tests for admin refund; subscriptions test doesn't cover refund path | Must add tests |
| Removed timeout handling | No test for timeout scenario | Must add test |
| `voidPayment` no longer exists | No test — will fail silently in production | Must add test |

### 4. Make a specific recommendation

The AI analysis gives you evidence for a clear recommendation:

```
## Regression Risk Assessment

### Must fix before merge
1. Update checkout.test.ts — processPayment signature changed, existing tests will fail to compile
2. Add refund tests — refundPayment return type changed, zero test coverage on admin and subscription callers
3. Add timeout polling test — old timeout handler is now dead code; new SDK returns 'pending' instead

### Should fix before merge
4. Remove or replace voidPayment call in admin portal — new SDK doesn't support it

### Test execution plan
- Run full payment test suite: test/payments/*.test.ts (14 tests, ~30s)
- Run checkout + subscription integration tests (8 tests, ~2min)
- Run admin portal E2E smoke tests (3 tests, ~5min)

### Confidence: HIGH
Analysis is based on concrete signature changes and caller tracing. Risks are specific and actionable.
```

### 5. Present to the developer

The output isn't a report — it's a review comment. Summarize in the PR:

> **Regression risk: HIGH.**
>
> - `processPayment` signature change breaks checkout tests (compile-time, caught immediately)
> - `refundPayment` return type change silently breaks admin refund path (runtime, no tests catch it)
> - `voidPayment` no longer exists in new SDK — admin portal calls it, will throw at runtime
> - Old timeout handler is dead code; new SDK doesn't throw on timeout
>
> Recommend: update checkout tests as part of this PR, add refund/void tests, and handle the timeout change before merging.

## When NOT to Use This Workflow

- **Trivial changes** (typo fixes, CSS tweaks, config bumps): The workflow overhead exceeds the risk. Use judgment.
- **Greenfield files**: No existing tests to affect. Use `test-generation.md` instead.
- **Dependency bumps without code changes**: The diff is the lockfile. The AI can't tell you what changed in the SDK. Use the package's changelog or release notes instead.

## Known Failure Modes

| Failure | Likely Cause |
|---------|-------------|
| AI misses a caller because it's dynamically imported | Dynamic imports aren't statically traceable — augment with `rg "processPayment"` across the codebase |
| Overstates risk for renamed private functions | If the function was `private` or `internal`, the blast radius is limited — check visibility markers before escalating |
| Lists every caller including dead code paths | The AI doesn't know which code paths are actually reachable. Cross-reference with code coverage data if available. |
| Confuses SDK method removals with renames | The AI infers from diff context. If the method was renamed (not removed), the "dead call" is actually a missed update. Check the SDK migration guide. |
