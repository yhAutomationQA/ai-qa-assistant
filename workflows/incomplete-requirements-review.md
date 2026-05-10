# Workflow: Reviewing Incomplete Requirements

## Scenario

The team gets a user story that's been through refinement but still has gaps:

> **As a** support agent
> **I want** to adjust an order's total after it's been placed
> **So that** I can issue courtesy credits without creating a new transaction

Acceptance criteria:
1. Agent can enter a positive or negative adjustment amount
2. Adjustment is logged in the order audit trail
3. Customer receives an email notification

This story has been "ready for dev" for two sprints. Nobody caught the gaps because everyone assumed they knew what "adjustment" means.

## Step-by-Step

### 1. Identify ambiguity points

Before AI: read the story and flag everything that's ambiguous in your own mind.

| Ambiguity | Your Question |
|-----------|--------------|
| "positive or negative" | Can the total go below zero? |
| "adjustment amount" | Is this a dollar amount or a percentage? |
| "logged in audit trail" | What fields are captured? |
| "email notification" | What does the email say? Can the agent preview it? |
| "courtesy credits" | Is this the same as refunds? Different accounting treatment? |

Now feed the story to AI with this task:

```
The following user story is going into development but has ambiguous or missing details that will cause bugs if not addressed before coding. For each ambiguity, explain (a) what specific question needs answering, (b) why it matters for testing, and (c) a concrete example of what could go wrong if it's left ambiguous.

User story: {{ story text }}
Acceptance criteria: {{ list }}
```

**What you'll get back:**

```
Ambiguity 1: "adjustment amount — dollar or percentage?"
- Question: Is the value in cents, dollars, or percentage points?
- Why it matters: A 10% discount vs $10 discount are completely different code paths. Tests for one will pass for the wrong reason if the other is implemented.
- Bug scenario: Agent enters "10" intending $10 off, but the system applies 10% off. $100 order → $90 instead of $90. Small difference here, but for a $2000 order: $1800 vs $1990.

Ambiguity 2: "Can total go below zero?"
- Question: Is there a floor of $0.00, or can the adjustment produce a negative total (negative balance)?
- Why it matters: Negative balances may require different invoicing, refund workflows, or tax recalculation.
- Bug scenario: Agent adjusts a $50 order by -$75. Total becomes -$25. The payment gateway rejects negative charges. Customer gets an email saying "your refund of -$25 has been processed" — confusing at best.

Ambiguity 3: "What triggers the email?"
- Question: Is the email sent on save, on confirmation dialog, or as a separate scheduled job?
- Why it matters: If the agent adjusts, saves a draft, edits again, then confirms — does the customer get 3 emails or 1?
- Bug scenario: Agent clicks "save" instead of "confirm". Customer gets an email about an adjustment that hasn't taken effect yet. Support gets a callback.

Ambiguity 4: "Who can adjust?"
- Question: Any support agent? Only senior agents? Is there a maximum adjustment amount per role?
- Why it matters: Authorization is a security boundary. Testing the wrong role/permission matrix is how PII leaks happen.
- Bug scenario: Junior agent adjusts a $5000 order by -$5000. No approval workflow. Order is essentially refunded by a single untrained click.

Ambiguity 5: "Audit trail — what's captured?"
- Question: Agent ID, before/after total, timestamp, reason code, IP address?
- Why it matters: Without specific fields, the audit trail might omit the one field needed for compliance.
- Bug scenario: Finance runs an audit report. They need "reason code" to categorize adjustments. The field doesn't exist because nobody specified it.
```

### 2. Triage: what to escalate

The AI identified 5 ambiguities. You decide which ones need PM input vs. which you can resolve by convention:

| Ambiguity | Action | Who |
|-----------|--------|-----|
| $ vs % | Must be answered — fundamentally different behavior | PM |
| Negative total | Must be answered — financial/legal boundary | PM |
| Email trigger | Can be resolved by looking at how existing notifications work | Dev + QA |
| Role permissions | Must be answered — security boundary | PM |
| Audit fields | Can be resolved from existing audit log schema | Dev |

### 3. Generate concrete examples with AI

For the ambiguities that need PM input, ask AI to produce concrete examples that force a decision:

```
For the following unresolved questions, produce 2-3 concrete examples that illustrate the tradeoffs. The goal is to give the PM enough context to make a decision, not to overwhelm them.

1. Dollar vs percentage: show a scenario where interpreting "10" as $10 vs 10% produces different outcomes.
2. Negative total: show a scenario where allowing negative totals causes downstream breakage vs. where it's legitimately useful.
3. Role permissions: show a scenario where a junior agent can cause harm vs. where restrictive permissions block legitimate work.
```

**Output:**

```
1. Dollar vs Percentage
   - Scenario: $2000 order, agent enters "10"
   - As dollars: new total = $1990
   - As percentage: new total = $1800
   - Difference: $190
   - Risk: Without clarifying, dev will pick whichever is easier to implement. If the wrong one ships, every courtesy credit is wrong.

2. Negative Total
   - Scenario: $50 order, adjustment of -$75
   - If clamped at $0: Customer gets $50 credit. Agent needs a separate process for the remaining $25.
   - If allowed to go negative: System shows -$25 balance. Payment gateway may reject downstream. Tax was already calculated on $50 — do we send a corrected invoice?
   - Tradeoff: Clamping is safer but may not cover all business cases.

3. Role Permissions
   - Scenario A: Junior agent adjusting a $5000 order by -$500. This might be legitimate (e.g., removing a mistaken line item).
   - Scenario B: Junior agent adjusting the same order by -$5000 (full refund). This exceeds their authority but the system allows it.
   - Tradeoff: A strict cap blocks legitimate work. No cap invites abuse. A soft cap with approval workflow is the middle ground.
```

### 4. Send structured feedback to product

Compile the analysis into a brief document. Format:

```
## Story: Order Adjustment — Gaps Found During Requirements Review

### Must resolve before dev starts
1. [ ] Adjustment unit — dollars, cents, or percentage? 
      Impact: wrong interpretation produces $190 error on a $2000 order.
2. [ ] Floor/ceiling — can total go below $0 or above original total?
      Impact: negative balances cause payment gateway failures.
3. [ ] Role-based limits — what's the max adjustment per role?
      Impact: junior agent could adjust $5000 without approval.

### Can resolve by convention (no PM decision needed)
4. [ ] Email trigger — match existing notification pattern (send on confirm, not on save)
5. [ ] Audit fields — align with existing audit log schema

### Suggested test scenarios (for reference)
- Adjust $2000 order by -$10 → total should be $1990 (assuming dollars)
- Adjust $50 order by -$75 → total should be $0 (assuming floor)
- Adjust $5000 as junior agent → should be rejected (assuming cap)
```

### 5. Don't skip the human review

The AI will surface gaps you missed, but it will also flag things that aren't actually gaps:

- **False positives**: "Is the adjustment synchronous or async?" — the story doesn't say, but for this system all adjustments are synchronous. The AI doesn't know that. Strike it.
- **False negatives**: The AI won't flag business logic gaps it can't infer. If the story omits an entire component (e.g., "what about tax recalculation?"), the AI probably won't catch it unless the omission is obvious from context.
- **Overconfidence**: The AI presents ambiguities with equal weight. Your job is to prioritize them for the PM.

## What This Replaces

This doesn't replace refinement. It replaces the silent head-nodding during sprint planning where everyone assumes "adjustment" means the same thing. The AI output is a forcing function for precision — PMs write tighter stories when they know someone will immediately flag every ambiguity.

## Known Failure Modes

| Failure | Likely Cause |
|---------|-------------|
| AI turns every word into an ambiguity | Prompt didn't specify "realistic risk of bug" — add a severity filter |
| Misses business logic gaps the team assumes | Domain knowledge the AI doesn't have — always supplement with your own review |
| Generates too many questions (PM resentment) | Cap the output: "list the 5 most impactful ambiguities" |
| Flagged questions are actually answered elsewhere in the spec | The prompt scope was too narrow — include the full spec, not just the story card |
