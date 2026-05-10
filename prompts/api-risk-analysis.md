# Prompt: API Risk Analysis

## When to Use

You have an API endpoint (or a candidate spec) and need a risk assessment to guide test coverage, determine shift-left priorities, or decide whether to add chaos engineering scenarios. This is for the *analysis* phase — the output is a risk matrix, not test code.

Works best for REST/gRPC APIs with defined schemas. GraphQL works too but requires specifying which queries/mutations are in scope.

## Input Template

---

**Endpoint**

```
{{ HTTP method }} {{ path }}
```

**Request schema** (or attach OpenAPI/Swagger snippet):

```json
{
  "parameters": { ... },
  "requestBody": { ... }
}
```

**Response schema**:

```json
{
  "200": { ... },
  "4xx": { ... },
  "5xx": { ... }
}
```

**Authentication & Authorization**: {{ e.g. "Bearer token, admin role required for POST, any authenticated user for GET" }}

**Dependencies**: {{ e.g. "reads from PostgreSQL, writes to SQS, calls external payment gateway" }}

**Service-level SLAs** (if documented):
- Latency p99: {{ ms }}
- Throughput: {{ req/s }}
- Consistency: {{ e.g. "read-after-write guaranteed" }}

**Existing test coverage** (if any): {{ link or summary }}

**Task**

Analyze the endpoint and produce a risk matrix.

For each risk, provide:

```
Risk ID: R##

Category: [data-integrity | auth | availability | correctness | performance | security | idempotency | consistency | dependency-failure]

Description: One sentence describing what could go wrong.

Trigger: What causes it? (specific input, state condition, concurrency scenario, downstream failure)

Detection: Can we detect this in tests? [easily / with effort / only in prod]

Severity: [critical / high / medium / low]
Likelihood: [high / medium / low]

Existing coverage: [covered / partial / none]
Suggested test approach:
- [approach 1]
- [approach 2]
```

Order the matrix by severity (highest first).

After the matrix, include a **One-Sentence Summary**:

> This endpoint's primary risk is {{ highest-risk summary }} because {{ reason }}.

---

## Example Output (abbreviated)

**Endpoint:** `POST /api/v2/orders`

```
R01 | data-integrity | Double charge on retry | Client retries with same idempotency key after timeout but first request succeeded | easily | critical | medium | none
     Suggested: Test idempotency with simulated timeout + retry

R02 | auth | Non-admin user creates order for another org | Missing org_id scoping check | easily | critical | low | partial
     Suggested: Test with valid token from different org asserts 403

R03 | dependency-failure | Payment gateway returns 500, order is left in 'pending' state forever | Gateway timeout mid-request | with effort | high | medium | none
     Suggested: Wiremock gateway to return 500 after partial write; verify compensation logic
```

**One-Sentence Summary:** This endpoint's primary risk is **data integrity under retry** because the payment flow is not idempotent and a client retry can produce duplicate charges.

## Known Failure Modes

| Failure | Likely Cause |
|---------|-------------|
| Risk analysis is too generic ("server could crash") | Endpoint context was too thin — add dependency details and SLA data |
| Over-indexes on security | Input schema was the only thing provided; add the business logic description |
| Detection column is always "only in prod" | That's sometimes correct — flag it but also suggest the closest test approximation |
| Suggests OWASP-level threats for an internal API | Clarify the threat model: internal vs external, authenticated vs public |
