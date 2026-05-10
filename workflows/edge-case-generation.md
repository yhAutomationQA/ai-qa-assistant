# Workflow: Generating Edge-Case Scenarios from a Feature

## Scenario

Product ships a spec for a "bulk user invitation" feature:

> Admins can upload a CSV of email addresses. The system validates each row and sends invitations to valid emails. Invalid rows are reported in a summary CSV that the admin can download.

That's it. No mention of file size limits, encoding, duplicate handling, partial failures, or concurrency. Your job is to find the gaps before the feature hits QA.

## Step-by-Step

### 1. Establish what we know

Write down the explicit constraints from the spec:

- Input: CSV file with email addresses
- Valid rows → invitation sent
- Invalid rows → reported in a downloadable summary CSV
- Actor: admin user

That's thin. The interesting work is in what's *not* said.

### 2. Apply systematic edge-case heuristics with AI

Feed the spec to the prompt with explicit heuristic categories:

---
**Feature:** Bulk user invitation by CSV upload
**Spec:** {{ spec text above }}
**Heuristics to apply:**
1. **Boundary analysis** — file sizes, row counts, email length limits, CSV column count
2. **Input encoding** — BOM, UTF-8 vs UTF-16, line endings (CRLF vs LF), null bytes
3. **Duplicate handling** — exact dupes, case-insensitive dupes, same email across multiple rows
4. **Partial failure** — 999 valid + 1 invalid: does the whole thing fail or just the bad row?
5. **Concurrency** — two admins upload the same list simultaneously
6. **State edge cases** — retry after partial upload, download summary before upload completes
7. **Security** — CSV injection (formulas starting with =, @, +, -), extremely long fields

For each heuristic, generate concrete test scenarios with specific inputs and expected outcomes.

**What you'll get back (abbreviated):**

```
## Boundary
- BC01: Empty CSV (0 rows) → reject with "file is empty" message
- BC02: 1-row CSV → process normally (minimum viable file)
- BC03: CSV with 10,000 rows → process within stated SLA or show progress indicator
- BC04: CSV with 10,001 rows → reject or paginate (spec doesn't specify a limit)

## Encoding
- ENC01: UTF-16 encoded CSV → detect and convert or reject with clear message
- ENC02: CSV with BOM → strip BOM, process normally
- ENC03: Mixed line endings (CRLF + LF) → parse correctly or normalize

## Duplicates
- DUP01: Same email appears twice → send one invitation, report duplicate in summary
- DUP02: "user@example.com" and "User@example.com" → are these the same? Spec doesn't say.
- DUP03: Email already exists as a user → skip, report as "already registered"

## Partial Failure
- PF01: 1000 valid + 1 invalid row → process the 1000, report the 1. Does not roll back.
- PF02: All rows invalid → no invitations sent, summary CSV contains all rows
- PF03: Network error after 500 of 1000 processed → what state is the system in?

## Concurrency
- CC01: Two admins upload same list → no duplicate invitations
- CC02: Admin uploads while another admin downloads a previous summary → no race condition

## Security
- SEC01: CSV cell starts with "=cmd|" or "=HYPERLINK()" → sanitize or reject
- SEC02: Email field contains newline or semicolon → validate as malformed email
```

### 3. Triage the output

Not all of these scenarios are worth testing. Apply your team's risk filter:

| Category | Keep? | Why |
|----------|-------|-----|
| BC01-04 | Keep | File size limits are a common miss; prod incidents often start here |
| ENC01-03 | Drop for now | Encoding issues are real but low-traffic; revisit if users report problems |
| DUP01-03 | Keep | Duplicate handling is explicitly undefined; this will bite us |
| PF01-03 | Keep | Partial failure behavior is the #1 source of "it looked like it worked" bugs |
| CC01-02 | Keep if time | Concurrency is high-risk but hard to test; at minimum document the known gap |
| SEC01-02 | Keep | CSV injection is well-known; quick win to add input sanitization |

You just turned 3 lines of spec into 6 validated test areas. Without AI, you'd get maybe 3 of these from experience — the AI catches the ones you'd miss because they're rare but devastating (encoding, concurrency, security).

### 4. Convert scenarios to test cases

Use the `test-generation.md` prompt with the kept scenarios. For each scenario, feed the concrete input and expected behavior. The result is a set of parameterized tests.

### 5. Surface the blind spots to product

Some scenarios uncovered questions only the PM can answer:

- **What's the max file size?** (BC04)
- **Are emails case-sensitive for dedup?** (DUP02)
- **Partial failure: roll back whole batch or process good rows?** (PF01)

Send these as a structured list back to product *before* writing automation. It's cheaper to change a spec than to rewrite tests.

## How Much of This Is Actually AI?

The heuristics (boundary, encoding, concurrency, etc.) are standard testing techniques. The AI's job isn't to invent them — it's to apply them faster than a human can type them out. The value is:

1. **Exhaustiveness**: AI doesn't get bored after the 4th scenario. It will list 20 edge cases while a human stops at 5.
2. **Speed**: 2 minutes of prompt engineering instead of 20 minutes of typing.
3. **Surface assumptions**: The AI will flag things you took for granted ("assumes UTF-8") because it's not making the same implicit assumptions.

## Known Failure Modes

| Failure | Likely Cause |
|---------|-------------|
| Scenarios are technically correct but irrelevant to your system | The heuristic list was too generic — constrain it with your system's actual constraints (e.g., "max upload 5MB", "emails are case-sensitive in our DB") |
| AI generates impossible scenarios ("test with 2^32 rows") | Heuristics didn't include practical bounds — add "realistic only" to the prompt |
| Misses domain-specific edge cases | Heuristics are generic — after AI output, add your own domain patterns (e.g., for fintech: rounding, currency conversion) |
| Output is too verbose to act on | Constrain the output format: ask for a table with columns "ID / Input / Expected Result / Priority" |
