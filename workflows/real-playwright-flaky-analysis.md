# Workflow: Analyzing a Flaky Playwright E2E Test

## Scenario

Your CI pipeline shows a Playwright test that fails ~3 out of 10 runs. The test checks that after a user submits a payment form, a confirmation receipt appears. Passes locally. Fails only in CI, seemingly at random. Every failure is a timeout waiting for the receipt text.

## Step-by-Step

### 1. Collect the artifacts

Before involving AI, gather:

- **Test source**: the full `it()` block that fails
- **Trace file**: `npx playwright show-trace trace.zip` — export the trace from CI
- **CI error output**: the full Playwright error message, not just the summary line
- **Pass/fail history**: last 15 CI runs for this specific test

Avoid the temptation to paste the entire CI log. You want: error message, trace actions leading to the failure, and the pattern of passes vs. failures.

### 2. Feed the `playwright-failure-analysis.md` prompt

Take the prompt template from `prompts/playwright-failure-analysis.md` and fill it with your artifacts. Include the trace events and the error message.

**What you'll get back:**

```
Root Cause Category: network
Confidence: medium
Evidence: Error is "locator.waitFor() timed out waiting for 'text=Receipt'".
Trace shows API request to /payments/confirm returned after the locator started.
Flaky: yes, likely trigger is API response timing variance on CI.

Suggested Fix:
- Option A (minimal): Add aria-role assertion on the loading spinner before waiting for receipt
- Option B (thorough): Use expect.poll() to wait for the receipt endpoint to return 200 before asserting DOM

One thing to try: Add --headed screenshot at failure point to see if the receipt rendered but wasn't scrolled into view.
```

### 3. Challenge the hypothesis

The AI said "network" — but the trace hasn't been examined yet. Open the trace:

- Look at the **Timeline** tab: is there a gap between the form submit action and the receipt locator?
- Check the **Network** tab: did the `/payments/confirm` call complete before the locator fired?
- Check **Console**: any uncaught promise rejections?

**Example finding from the trace:**

The trace shows the form submission succeeded and the API returned 200 within 200ms. But the receipt component re-renders twice — once with a skeleton placeholder, once with the actual data. The locator `page.getByText('Receipt')` matched the skeleton text on the first render (hidden), then failed when the skeleton was replaced because the locator had already resolved to the detached element.

**This contradicts the AI's network hypothesis.** The actual root cause is a **stale element** — the locator resolved too early to a skeleton that got replaced.

### 4. Refine with the trace evidence

Add the trace findings to a follow-up:

```
The AI classified this as "network" but the trace shows the API completed in 200ms.
The receipt element re-renders twice: skeleton then real data.
page.getByText('Receipt') resolves on the first render, then the element is detached.

Reclassify based on this evidence.
```

**Refined output:**

```
Root Cause Category: stale-element
Confidence: high
Evidence: Trace confirms API completed on time. Receipt DOM goes skeleton → data.
Locator resolves on skeleton instance; element detaches on re-render.
Fix: Use getByTestId('receipt-content') that only appears on the data render,
or wait for the skeleton to disappear first.
```

### 5. Fix and validate

The real fix (in this case):

```typescript
// Before
await expect(page.getByText('Receipt')).toBeVisible();

// After
await expect(page.getByTestId('receipt-content')).toBeVisible();
// getByTestId targets the data-rendered element, not the skeleton placeholder
```

Run the test 20 times in CI. If it passes 20/20, close. If not, return to step 3.

## Why the AI Got It Wrong (and Why That's OK)

The AI guessed "network" because the error message was a timeout, and CI timeouts are often network-related. It didn't have the trace data — only the error message and pass/fail pattern. The trace was the decisive artifact.

This is the normal pattern: AI generates a hypothesis; the engineer tests it against reality. The value isn't the first guess — it's that the structured analysis forced a specific, testable hypothesis instead of a vague "it's flaky, let's add a retry."

## Anti-Patterns

| Anti-pattern | Why It Fails |
|---|---|
| Pasting the whole CI log and asking "why did this fail?" | Produces vague analysis; the AI can't distinguish signal from noise in 2000 lines of log. Curate the input. |
| Accepting the first AI answer without checking the trace | Trace data is the single most reliable artifact for Playwright failures. AI without trace is guessing. |
| Asking AI to rewrite the test after every failure | You train the test to be resilient to bugs, not resilient to flakiness. Fix the root cause, not the symptom. |
| Using `page.waitForTimeout(n)` as a fix | Masks the flakiness, slows the suite, makes the next failure harder to diagnose. The prompt template bans this for a reason. |
