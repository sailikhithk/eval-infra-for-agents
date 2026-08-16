# Chapter 2: Ground Truth - Building a Versioned Sample Store

> How to build a versioned ground-truth sample store with PII redaction, metadata, and
> reproducibility. Based on the 1,690-sample store at Airbnb BPI Virtual Analyst.

## Why ground truth is the foundation

Every evaluation is only as good as its ground truth. If your labels are wrong, stale, or
inconsistent, your accuracy metrics are meaningless. You will optimize for the wrong thing.

The most common mistake teams make is treating ground truth as a one-time task: label some
samples, put them in a file, and reuse them forever. This leads to eval drift: the labels
become stale as the agent evolves, and you keep optimizing against labels that no longer
reflect what the agent should be doing.

The solution is a versioned ground-truth store: a structured, versioned, PII-safe collection
of samples that you can update independently of the agent versions you evaluate against it.

## The structure

Each sample in the store has:

```python
{
    "id": "bpi-va-0001",
    "version": "v3",
    "input": "Show me the top 5 routes by booking volume in Q1 2026",
    "expected_output": {
        "intent": "route_analytics",
        "filters": {"metric": "booking_volume", "time_period": "Q1_2026", "top_n": 5},
        "visualization": "bar_chart"
    },
    "metadata": {
        "category": "analytics",
        "difficulty": "easy",
        "source": "real_user_query",
        "date_labeled": "2026-01-15",
        "labeler": "analyst_01",
        "pii_redacted": true
    }
}
```

### Key fields

- **id**: Unique identifier, never changes. Lets you track a sample across versions.
- **version**: The ground-truth version this label belongs to. When you relabel, you create
  a new version, not overwrite.
- **input**: The user query or task. PII-redacted before any LLM submission.
- **expected_output**: The ground-truth label. Structure depends on your agent's task.
- **metadata**: Category, difficulty, source, date labeled, labeler, PII status.

## Versioning

Ground-truth versions are independent of agent versions. This is critical:

```
Ground truth versions: v1, v2, v3
Agent versions: agent-001, agent-002, ..., agent-023

You can run:
  agent-001 against v1  (historical comparison)
  agent-023 against v3  (current comparison)
  agent-023 against v1  (regression check: did we break old cases?)
```

When you relabel, you create a new ground-truth version. The old version is preserved. This
lets you answer: "Did agent-023 perform better than agent-001 on the same ground truth?"

## PII redaction

Before any sample is sent to an LLM (for eval or for production), PII must be redacted. At
BPI VA, we use Microsoft Presidio with 12 custom entity types:

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

class PIIRedactor:
    def __init__(self):
        self.analyzer = AnalyzerEngine()
        self.anonymizer = AnonymizerEngine()
        # 12 entity types: PERSON, EMAIL, PHONE, LOCATION, DATE_TIME,
        # IBAN, IP_ADDRESS, URL, US_SSN, US_DRIVER_LICENSE, NRP, MEDICAL_LICENSE
        self.entities = [
            "PERSON", "EMAIL_ADDRESS", "PHONE_NUMBER", "LOCATION",
            "DATE_TIME", "IBAN_CODE", "IP_ADDRESS", "URL",
            "US_SSN", "US_DRIVER_LICENSE", "NRP", "MEDICAL_LICENSE"
        ]

    def redact(self, text: str) -> str:
        results = self.analyzer.analyze(
            text=text,
            entities=self.entities,
            language="en"
        )
        redacted = self.anonymizer.anonymize(
            text=text,
            analyzer_results=results
        )
        return redacted.text
```

### The singleton pattern

At scale, creating a new `AnalyzerEngine` for every request is expensive. We use a batch
analyzer singleton that is 30% faster than the default configuration:

```python
class BatchPIIRedactor:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._analyzer = AnalyzerEngine()
            cls._instance._anonymizer = AnonymizerEngine()
        return cls._instance

    def redact_batch(self, texts: list[str]) -> list[str]:
        # Batch analysis is faster than per-text analysis
        # because the NLP model is loaded once
        results = [
            self._analyzer.analyze(
                text=text,
                entities=self._entities,
                language="en"
            )
            for text in texts
        ]
        return [
            self._anonymizer.anonymize(
                text=text,
                analyzer_results=result
            ).text
            for text, result in zip(texts, results)
        ]
```

## Storage

At BPI VA, the ground-truth store is stored as JSON files in a versioned directory structure:

```
ground_truth/
  v1/
    samples.jsonl      # 1,200 samples
    metadata.json      # version metadata, labeler info, date
  v2/
    samples.jsonl      # 1,450 samples (added 250 new)
    metadata.json
  v3/
    samples.jsonl      # 1,690 samples (added 240 new, relabeled 50)
    metadata.json
```

Each `samples.jsonl` file is JSON Lines format (one JSON object per line), which is efficient
for streaming and parallel processing.

## Sampling strategy

Not all samples are equal. A good ground-truth store has:

- **Real user queries** (60%): Actual queries from production, PII-redacted. These are the
  most valuable because they reflect real usage.
- **Edge cases** (20%): Manually crafted cases that test specific failure modes.
- **Adversarial cases** (10%): Cases designed to break the agent (ambiguous inputs, missing
  context, conflicting constraints).
- **Regression cases** (10%): Cases that previous agent versions failed on. These ensure
  that fixes do not regress.

## Labeling process

1. Collect candidate samples from production logs (PII-redacted)
2. Have 2 labelers independently label each sample
3. Flag disagreements for review
4. Resolve disagreements with a third labeler or domain expert
5. Add resolved labels to the next ground-truth version

## The eval harness reads the store

```python
class GroundTruthStore:
    def __init__(self, version: str, path: str = "ground_truth"):
        self.version = version
        self.path = f"{path}/{version}/samples.jsonl"
        self._samples = None

    @property
    def samples(self) -> list[dict]:
        if self._samples is None:
            self._samples = []
            with open(self.path) as f:
                for line in f:
                    self._samples.append(json.loads(line))
        return self._samples

    def filter_by_category(self, category: str) -> list[dict]:
        return [s for s in self.samples if s["metadata"]["category"] == category]

    def filter_by_difficulty(self, difficulty: str) -> list[dict]:
        return [s for s in self.samples if s["metadata"]["difficulty"] == difficulty]
```

## Common mistakes

1. **No versioning**: Overwriting labels instead of creating new versions. You lose the
   ability to compare agent versions against the same ground truth.
2. **No PII redaction**: Sending raw user queries to LLMs for eval. This is a compliance
   violation and a security risk.
3. **No metadata**: Samples without category, difficulty, or source. You cannot do
   per-category analysis or identify weak areas.
4. **Too few samples**: Evaluating on 50 samples. You need 1,000+ for statistical
   significance on per-category metrics.
5. **No adversarial cases**: Only testing happy-path queries. Your agent will fail on the
   first adversarial input it sees in production.

## Summary

A versioned ground-truth store is the foundation of production-grade eval infrastructure.
It enables reproducible comparison across agent versions, PII-safe evaluation, and
per-category analysis. The 1,690-sample store at BPI VA took 6 months to build and is the
single most valuable asset in the agent quality program.

---

Previous: [Chapter 1: The Eval Problem](01-the-eval-problem.md)
Next: [Chapter 3: The FacadeDriver Pattern - Multi-LLM Orchestration](03-facadedriver.md)
