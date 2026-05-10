# Workflow: Flaky Test Triage

Goal: Classify a flaky test, identify the root cause category, and produce a fix hypothesis — in under 10 minutes.

## Inputs Needed

- Test source code
- Last 10 run results (pass/fail timestamps + any error output)
- CI logs from a failing run (ideally the full stdout/stderr)

## Steps

### 1. Collect run history

Extract the pass/fail pattern. Key questions:
- Does it fail on every Nth run or randomly?
- Does it correlate with a specific node/worker?
- Does it fail only in CI, or locally too?

Output: a short pattern description like *"fails ~20% of runs, always on CI worker 3, never locally."*

### 2. Classify flakiness type

Feed the source + pattern to AI with this query:

```
Test source: {{ test code }}
Run history: {{ pattern from step 1 }}
CI error excerpt: {{ relevant lines from failing log }}

Classify the flakiness into one of:
- Async/timing: depends on wall clock, network, or thread scheduling
- Shared state: test mutates global state (DB, FS, env vars) not cleaned up
- Order dependency: passes alone, fails when run after a specific other test
- External service: relies on a real HTTP endpoint, queue, or DB that is flaky
- Probabilistic: uses random data, dates, or hash-based ordering that rarely collides

Return: classification, a 2-sentence justification, and which lines in the source are most likely involved.
```

### 3. Fix hypothesis

- If **async/timing**: propose replacing sleep() with a wait/retry loop, or adding a barrier.
- If **shared state**: propose isolating state per test (fresh DB transaction, env var sandbox, tempdir).
- If **order dependency**: propose using pytest-randomly or randomized test order to detect it, then fix the shared state.
- If **external service**: propose wiremock/mockserver or a dedicated test fixture.
- If **probabilistic**: propose freezing the seed or using property-based testing with explicit shrinking.

### 4. Human review

- Is the classification correct? (It's wrong ~30% of the time — trust but verify.)
- Does the fix introduce more flakiness? (E.g., adding sleeps "fixes" one test but slows the suite.)
- Can we write a regression test that would catch this flakiness deterministically?

### 5. Fix + validate

Commit the fix. Then run the test 20 times in CI (or locally with a loop). If it passes 20/20, mark as resolved. If not, return to step 2 — the classification was wrong.

## Decision Tree

```
Fails in CI only?
├── Yes → Check for env-specific state (env vars, clock skew, resource limits)
├── No  → Runs locally?
│       ├── No  → Likely async/timing or probabilistic
│       └── Yes → Dependency on machine state that differs (seed data, port availability)

Failure count correlates with parallel workers?
├── Yes → Shared state via global fixture or concurrency bug
└── No  → Look at individual failure messages for pattern
```

## One-Page Cheat Sheet

```
PATTERN                         LIKELY CLASSIFICATION        FIX APPROACH
fails only on Mondays           Time-dependent logic        Freeze date in test
fails on CI worker 5 always     Node-specific state         Check worker labels/env
fails after test_booking_*      Order dependency            pytest-randomly + isolate state
passes 9/10 times locally       Async/timing                Replace sleep with wait_until()
fails with ConnectionRefused    External service            Mock the dependency
fails with "expected X got X"   Probabilistic noop          Check for unordered collections
