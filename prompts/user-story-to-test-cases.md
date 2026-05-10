# Prompt: User Story to Test Cases

## When to Use

You have a user story in standard format ("As a..., I want..., So that...") and need a structured breakdown into testable scenarios. This is for the *analysis* phase — generating test cases, not test code. Use the output as input to `test-generation.md` when you're ready to write automation.

## Input Template

---

**User Story**

```
As a {{ role }},
I want {{ feature/goal }}
So that {{ benefit/value }}
```

**Acceptance Criteria** (if any):

```
{{ paste existing AC here, or leave blank }}
```

**Technical Context**
- System/Service: {{ name }}
- Known constraints: {{ e.g. "must work offline", "rate-limited to 10 req/s", "async with eventual consistency" }}
- Existing test coverage: {{ what's already tested around this area }}

**Task**

Decompose this user story into test cases. For each case, provide:

| # | Scenario | Given | When | Then | Priority | Risk |
|---|----------|-------|------|------|----------|------|
|  |  |  |  |  | P0/P1/P2 | High/Med/Low |

Prefer the **Gherkin-style columns** above. Avoid prose paragraphs — the output should be scannable.

Cover these categories, in order:
1. **Functional happy path** — the core value of the story works
2. **Functional variants** — different user roles, data sizes, input formats
3. **Error & rejection** — validation failures, permission denials, system errors
4. **Boundary & edge** — empty states, max values, concurrency, timeouts
5. **Non-functional hints** — anything in the story that implies performance, accessibility, or reliability

**Output Format**

A markdown table. No commentary before or after. If a category has zero cases, write a row with "None identified" in the Scenario column.

---

## Example

**Input:**

```
As a warehouse manager,
I want to generate a daily restock report,
So that I can reorder low-stock items before they run out.
```

**Output:**

| # | Scenario | Given | When | Then | Priority | Risk |
|---|----------|-------|------|------|----------|------|
| 1 | Standard report generation | items exist with stock < threshold | report is requested | CSV/PDF is returned with correct item list | P0 | High |
| 2 | No low-stock items | all items have stock > threshold | report is requested | empty report with "All items sufficiently stocked" message | P1 | Low |
| 3 | Report with mixed stock levels | items below, at, and above threshold | report is requested | only items below threshold appear in output | P0 | Med |
| 4 | Warehouse with zero inventory | warehouse exists but no items recorded | report is requested | empty report, no crash | P1 | Med |
| 5 | Concurrent report requests | two managers request simultaneously | both requests in flight | both receive correct reports, no data race | P2 | High |
| 6 | Performance: 10,000+ items | warehouse with 10,000 items, 5,000 low-stock | report is requested | renders within 5 seconds | P2 | Med |
| 7 | Unauthorized access | non-manager user attempts request | request submitted | 403 Forbidden | P0 | High |

## Known Failure Modes

| Failure | Likely Cause |
|---------|-------------|
| Output is too abstract ("test that it works") | Story lacks concrete details — add acceptance criteria or constraints |
| Missing error cases | Story is framed only as a success scenario; deliberately ask for failure cases as a follow-up |
| Priorities are all P0 | AI doesn't know your risk model — override priorities manually; they're a starting guess |
| Edge cases feel generic ("test with null input") | Story doesn't describe the data model — provide the schema or type definitions |
