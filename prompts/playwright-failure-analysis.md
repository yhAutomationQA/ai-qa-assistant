# Prompt: Playwright Failure Analysis

## When to Use

A Playwright test failed in CI. You have the error message, trace file, and/or screenshot. You need a root-cause hypothesis before deciding whether to fix the test, the app, or the infrastructure.

This targets **flaky E2E failures** — the ones where the test sometimes passes, sometimes doesn't. Deterministic failures (app is actually broken) are better served by standard bug reporting.

## Input Template

---

**Test Identity**
- Test file: {{ path }}
- Test name: {{ describe + it block name }}
- Browser: {{ chromium / firefox / webkit }}
- Run mode: {{ headed / headless / mobile emulation }}

**Failure Artifacts**

Error message(s):
```
{{ paste the full Playwright error — including the diff if snapshot-based }}
```

Screenshot description (or attach): {{ what's visible? loaded state? spinner? error toast? blank screen? }}

Trace events leading to failure:
```
{{ paste 10-15 lines from trace/log before the failure }}
```

**Test Source** (relevant portion):

```typescript
{{ paste the test code, or at minimum the locator + action that failed }}
```

**Previous runs (last 10):**
- {{ pass/fail pattern, e.g. "F, F, P, P, F, P, P, P, P, F" }}

**Task**

Analyze the failure and return a structured assessment:

```
Root Cause Category: [timing / stale-element / network / state-leak / locator / infra / real-bug]
Confidence: [high / medium / low]
Evidence: [1-3 sentences tying the error message / trace / source to the category]

Flaky or Deterministic: [flaky / deterministic / uncertain]
If flaky: most likely trigger (race condition, order dependency, network jitter, etc.)

Suggested Fix:
- Option A (minimal): [short description, estimated effort]
- Option B (thorough): [short description, estimated effort]

One thing to try before fixing:
[re-run with --retries=0, capture HAR, dump page HTML at failure point, etc.]
```

Do NOT suggest increasing timeouts or adding `page.waitForTimeout(n)` as a fix. If that's all you have, say "no good fix identified."

---

## Output Quick-Reference

```
CATEGORY          SIGNAL                          CONFIDENCE BOOSTER
timing            "locator.waitFor() timed out"   Failure disappears with --headed
stale-element     "element is detached from DOM"  Page re-rendered between locator and action
network           "net::ERR_CONNECTION_RESET"     Fails only on CI, correlates with API latency
state-leak        test fails only after test N    Swap test order locally to verify
locator           "strict mode violation"         Multiple matches; inspect page DOM
infra             "Browser closed unexpectedly"   Correlates with high CI concurrency or OOM
real-bug          reproduces first try, always    File a bug report; this isn't a test problem
```

## Known Failure Modes

| Failure | Likely Cause |
|---------|-------------|
| AI blames timing for every failure | Timing is the default guess when evidence is thin — push for a second category if confidence is low |
| Suggests `page.waitForTimeout(2000)` | Output format constraint wasn't clear; re-send with the anti-waitForTimeout instruction |
| Can't distinguish locator vs. real-bug | Comparing the error with the page state (screenshot) is the only reliable way; without a screenshot, accept lower confidence |
| Over-fixes with `toBeVisible` on every element | Remind the AI: prefer locator actions that wait by default, avoid pre-flight assertions |
