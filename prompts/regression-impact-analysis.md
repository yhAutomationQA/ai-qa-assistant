# Prompt: Regression Impact Analysis

## When to Use

A pull request modifies existing code. You need to identify which tests are most likely affected, which coverage gaps exist, and where the risk of undetected regression is highest. Run this during PR review, before merging.

## Input Template

---

**PR Metadata**
- Title: {{ title }}
- Description: {{ description }}
- Files changed: {{ list of changed file paths }}
- Lines added / removed: {{ +N / -M }}

**Git Diff** (or summarize the changes):

```diff
{{ paste the diff, or key hunks if it's large }}
```

**Test Suite Map** (relevant subset):

| Test File | What It Covers | Last Green Run |
|-----------|---------------|----------------|
| {{ path }} | {{ description }} | {{ date }} |
| {{ path }} | {{ description }} | {{ date }} |

**Code Dependencies** (direct imports / usages of changed modules):

```
{{ modules/packages that import the changed code }}
```

**Task**

Return a structured analysis:

```
## Impact Summary
Changes to {{ module }} affect {{ count }} test files directly and {{ count }} indirectly.

## Directly Affected Tests
| Test | File | Risk Level | Why |
|------|------|-----------|-----|
| {{ name }} | {{ path }} | high/med/low | {{ 1-sentence reason }} |

## Indirectly Affected Tests
| Test | File | Risk Level | Dependency Chain |
|------|------|-----------|-----------------|
| {{ name }} | {{ path }} | high/med/low | {{ e.g. A → B → changed C }} |

## Coverage Gaps
| Gap | Risk | Suggested New Test |
|-----|------|-------------------|
| {{ what's not tested }} | high/med/low | {{ brief description }} |

## Recommendation
- [ ] Run full {{ area }} test suite ({{ N }} tests, ~{{ M }} min)
- [ ] Run targeted subset ({{ N }} tests listed above)
- [ ] Add {{ N }} new tests before merging (high-risk gaps only)
- [ ] No additional testing needed (changes are safe)

## Confidence Assessment
Confidence in this analysis: [high / medium / low]
If low, the missing information is: {{ what would help }}
```

Do NOT suggest running the entire project test suite unless the change touches a foundational shared module. Be specific about which subset to run.

---

## Example

**Input:** A PR that changes the `calculateShipping` function in `src/pricing/shipping.ts`.

**Output (abbreviated):**

```
## Impact Summary
Changes to `calculateShipping` affect 4 test files directly and 2 indirectly.

## Directly Affected Tests
| Test | File | Risk | Why |
|------|------|------|-----|
| calculateShipping handles weight tiers | tests/pricing/shipping.test.ts | high | Core logic changed |
| calculateShipping handles free shipping threshold | tests/pricing/shipping.test.ts | high | Edge case threshold modified |
| order total includes shipping | tests/pricing/order.test.ts | med | Imports calculateShipping |

## Coverage Gaps
| Gap | Risk | Suggested Test |
|-----|------|---------------|
| No test for shipping with zero-weight items | medium | calculateShipping with weight=0 returns $0 or min rate |
| No test for international shipping rates | low | Only if this PR touches region detection |

## Recommendation
- [x] Run targeted subset (6 tests listed above)

## Confidence
Confidence: high
```

## Known Failure Modes

| Failure | Likely Cause |
|---------|-------------|
| Flags every test in the repo as affected | Change is in a shared utility — accept the recommendation to run full suite |
| Misses integration/E2E tests that exercise the changed code | Test map didn't include E2E suites — add them as rows in the input |
| Recommends tests that don't exist (e.g. suggests checking a non-existent fixture) | The AI hallucinated the test infrastructure — always verify test paths exist before acting |
| Low confidence when it should be high | Large diff — split the input into one analysis per changed module for clearer results |
