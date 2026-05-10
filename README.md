# AI Quality Engineering Lab

An internal experiment in practical AI-assisted QA workflows. Not a product. Not a chatbot. A toolkit for understanding where AI actually helps (and where it doesn't).

## Problem

QA engineers spend ~40% of their time on activities that feel automatable: writing boilerplate tests, triaging flaky results, scanning regression logs. LLMs can help, but it's not obvious *how* without overpromising or shipping useless abstractions.

This repo is where we try things, break them, and document what survives.

## Focus Areas

| Area | Signal |
|------|--------|
| **Test generation** | Translate acceptance criteria → executable tests. Works best with structured input. |
| **Bug analysis** | Feed crash logs, stack traces, or reproduction steps → hypothesis + minimal reproduction. |
| **Regression review** | Diff-aware analysis — given a PR diff and test results, flag missing coverage. |
| **Exploratory testing** | Generate session charters, edge-case prompts, and oracles from feature descriptions. |
| **Flaky test investigation** | Analyze test history + source code to classify flakiness type and suggest fixes. |

## Structure

```
├── prompts/            # Reusable prompt templates (markdown, structured)
├── workflows/          # Step-by-step processes combining AI + human review
├── lab/                # Lab notebook — experiment logs, failures, observations
│   └── experiments/    # One file per experiment, dated
```

## How to Use

1. Browse `prompts/` for a relevant template
2. Read the corresponding `workflows/` doc for process
3. Run the experiment, log results in `lab/experiments/`
4. If it works, refine. If it doesn't, log why.

## Principles

- Prompts are **structured**, not conversational — we're engineering inputs, not chatting.
- Human always reviews before commit. AI generates drafts, hypotheses, and analysis. Humans decide.
- Prefer **deterministic tooling** where possible. Use AI only where it adds non-trivial value.
- If a workflow requires more than 3 steps, it's too complicated. Simplify or scrap it.
