# Chapter 1: The Eval Problem

> Why ad hoc evaluation fails at scale, and what production-grade eval infrastructure looks like.

## The problem

You shipped an LLM agent to production. It works. Users are happy. Then you change the prompt.

Was the change good or bad?

If you are like most teams, you have no systematic way to answer this question. You:
1. Run a few manual tests on examples you remember
2. Ask a colleague to "take a look"
3. Ship it and watch for complaints

This is ad hoc evaluation. It works when you have 10 users and 1 agent version. It breaks
when you have 55+ analysts, 23+ agent versions, and 1,690 ground-truth samples that need to
be re-evaluated every time you change a prompt, swap a model, or update a retrieval pipeline.

## What goes wrong without eval infrastructure

I have seen all of these failures in production:

### Silent regressions

You change a prompt to fix one failure mode. The fix works. But it introduces a new failure
mode on a different class of inputs that you did not test. The regression reaches production.
A user notices. You did not.

### Model swap surprises

You swap GPT-4o for Claude 3.5 Sonnet because it is cheaper. Overall accuracy stays the same.
But 8% of cases flip from correct to incorrect, and 6% flip from incorrect to correct. The
net is positive, but the 8% regression includes your most important use cases. Without
flip-analysis, you never see this.

### Cost surprises

You add a new LLM provider (Perplexity, say) to your orchestration layer. Costs look fine in
the dashboard. But the dashboard is underreporting Perplexity costs by 10x because
`num_search_queries` is not being propagated to the field that the cost calculator reads.
You find out when the monthly bill arrives. (This is the bug that LangChain PR #39351 fixed.)

### Eval drift

Your ground-truth samples were labeled 6 months ago. The agent has evolved. The labels are
still "correct" but they no longer reflect what the agent should be doing. You keep
optimizing against stale labels. Accuracy goes up. Quality goes down.

### The "it worked on my machine" problem

Your eval runs locally on 50 samples. It passes. You deploy. It fails on 10,000 rows in
production because the batching, PII redaction, and rate-limiting behave differently at
scale. Your local eval did not test the production pipeline.

## What production-grade eval infrastructure looks like

At Airbnb's BPI Virtual Analyst, the eval infrastructure has these components:

### 1. Versioned ground-truth store

1,690 samples, each with:
- A unique ID and version number
- An input (the user query or task)
- An expected output (the ground-truth label)
- Metadata (category, difficulty, source, date labeled)
- PII redaction applied before any LLM submission

The store is versioned. When the labeling criteria change, you create a new version, not
overwrite the old one. This lets you compare agent versions against the same ground truth
over time.

### 2. Agent version registry

Every agent version is registered with:
- A version ID
- The prompt template hash
- The model configuration (provider, model, temperature, etc.)
- The retrieval pipeline configuration
- The date it was created

This lets you run any agent version against any ground-truth version, reproducibly.

### 3. Evaluation harness

A harness that:
- Takes an agent version and a ground-truth version
- Runs the agent against every sample in the ground-truth store
- Collects the agent's output, latency, token usage, and cost
- Compares the output to the expected output using a scoring function
- Produces a report: overall accuracy, per-category accuracy, latency distribution, cost

### 4. Dual-model A/B comparison

A comparison tool that:
- Runs two agent versions against the same ground-truth version
- Identifies cases where the two versions disagree (flips)
- Classifies each flip as "correct to incorrect" or "incorrect to correct"
- Produces a flip-analysis report

This is the most important tool. Overall accuracy can stay the same while important cases
flip. Flip-analysis catches this.

### 5. Cost tracking

Accurate per-provider, per-model cost tracking that:
- Reads token usage from the LLM response
- Applies the correct pricing for each provider
- Handles provider-specific cost factors (e.g., Perplexity's `num_search_queries`)
- Aggregates costs across the eval run

### 6. Production pipeline parity

The eval harness runs the same pipeline that production uses:
- Same batching and streaming logic
- Same PII redaction (Microsoft Presidio, 12 entity types)
- Same rate limiting and retry logic
- Same LLM orchestration (FacadeDriver, 30+ models)

This eliminates the "it worked on my machine" problem. If the eval passes, production will
behave the same way.

## The eval-first workflow

With this infrastructure, the workflow becomes:

1. Make a change (prompt, model, retrieval, pipeline)
2. Register a new agent version
3. Run the eval harness against the current ground-truth version
4. Review the report: overall accuracy, per-category accuracy, flips, cost
5. If accuracy improved and no critical flips: ship to production
6. If accuracy regressed or critical flips: investigate and fix

This workflow scales. It works for 1 change or 100 changes. It catches regressions before
they reach production. It gives the team confidence to ship.

## What this handbook covers

The remaining chapters walk through each component in detail, with code examples based on
production systems:

- Chapter 2: Building a versioned ground-truth store
- Chapter 3: The FacadeDriver pattern for multi-LLM orchestration
- Chapter 4: Dual-model A/B comparison and flip analysis
- Chapter 5: Cost tracking at scale (the LangChain #39351 fix and beyond)
- Chapter 6: Reliability scoring after call end (the LiveKit #6754 pattern)
- Chapter 7: PII-safe batching and streaming (Microsoft Presidio)
- Chapter 8: Scaling from 600 to 10,000 rows per run
- Chapter 9: Building an agent quality program
- Chapter 10: Open-source eval tools (Ragas, LangSmith, Braintrust)
- Chapter 11: Test-first agent development
- Chapter 12: Production checklist: shipping an agent safely

Each chapter is a focused 8-10 minute read with companion code notebooks. The code is
simplified but structurally faithful to what runs in production.

---

Next: [Chapter 2: Ground Truth - Building a Versioned Sample Store](02-ground-truth.md)
