# AI Quality Engineering Lab

An internal experiment in practical AI-assisted QA workflows. Not a product. Not a chatbot. A toolkit for understanding where AI actually helps (and where it doesn't).

## Problem

QA engineers spend ~40% of their time on activities that feel automatable: writing boilerplate tests, triaging flaky results, scanning regression logs. LLMs can help, but it's not obvious *how* without overpromising or shipping useless abstractions.

This repo is where we try things, break them, and document what survives.

## Focus Areas

| Area | Signal |
|------|--------|
| **Test generation** | Translate acceptance criteria or user stories → executable tests. Works best with structured input. |
| **Bug analysis** | Feed crash logs, stack traces, or reproduction steps → hypothesis + minimal reproduction. |
| **Regression review** | Diff-aware analysis — given a PR diff and test suite map, flag missing coverage and blast radius. |
| **Exploratory testing** | Generate session charters, edge-case prompts, and oracles from feature descriptions. |
| **Flaky test investigation** | Analyze test history + trace artifacts to classify flakiness type and suggest fixes. |
| **Requirements analysis** | Detect ambiguities in user stories before they become bugs in production. |
| **AI governance** | Rules for when to trust (and not trust) AI-generated test automation. |

## Structure

```
├── prompts/                        # Reusable prompt templates (structured, not conversational)
│   ├── test-generation.md          # Acceptance criteria → test file
│   ├── user-story-to-test-cases.md # User story → Gherkin test case table
│   ├── playwright-failure-analysis.md
│   ├── api-risk-analysis.md        # Endpoint spec → risk matrix
│   └── regression-impact-analysis.md
│
├── workflows/                      # Step-by-step processes combining AI + human review
│   ├── flaky-test-triage.md        # General flaky test classification
│   ├── real-playwright-flaky-analysis.md  # Playwright trace-driven investigation
│   ├── edge-case-generation.md     # Systematic edge case discovery from specs
│   ├── incomplete-requirements-review.md  # Ambiguity detection in user stories
│   ├── regression-risk-identification.md  # PR diff blast radius analysis
│   └── ai-governance.md            # When to trust AI output, validation requirements
│
├── lab/                            # Experiment logs, failures, observations
│   ├── index.md                    # Lab notebook
│   └── experiments/
│       └── _template.md            # Reproducible experiment format
```

## How to Use

1. Browse `prompts/` for a template matching your task
2. Read the corresponding `workflows/` doc for the full process
3. Run the experiment, log results in `lab/experiments/`
4. Consult `workflows/ai-governance.md` before merging AI output into the codebase
5. If it works, refine. If it doesn't, log why.

## Principles

- Prompts are **structured**, not conversational — we're engineering inputs, not chatting.
- Human always reviews before commit. AI generates drafts, hypotheses, and analysis. Humans decide.
- Prefer **deterministic tooling** where possible. Use AI only where it adds non-trivial value.
- If a workflow requires more than 3 steps, it's too complicated. Simplify or scrap it.
- A test that compiles but doesn't catch bugs is worse than no test at all.
