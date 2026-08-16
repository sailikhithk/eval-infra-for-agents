# Chapter 3: The FacadeDriver Pattern - Multi-LLM Orchestration

> How to orchestrate 30+ LLM integrations behind a single unified interface.
> Based on the FacadeDriver abstraction at Airbnb BPI Virtual Analyst.

## The problem

You need to call multiple LLM providers in production:

- Azure OpenAI (GPT-4o, GPT-4.1, o1, o3)
- AWS Bedrock (Claude 3.5 Sonnet, Claude 4.5 Sonnet, Haiku)
- Google Vertex AI (Gemini 2.0 Flash, Gemini 2.5 Pro)
- vLLM-hosted open models (Qwen, Llama, Mixtral)
- Perplexity (sonar-reasoning-pro)
- Maybe more

Each provider has:
- A different SDK
- A different authentication method
- A different request/response format
- A different token counting method
- A different cost calculation
- A different rate limit
- A different streaming protocol

Without abstraction, your application code becomes a mess of if-else branches, each handling
a different provider's quirks. Adding a new provider means touching every call site. Testing
is a nightmare because you need credentials for every provider.

## The FacadeDriver pattern

The FacadeDriver pattern puts a single unified interface in front of all providers:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

@dataclass
class LLMResponse:
    content: str
    model: str
    provider: str
    input_tokens: int
    output_tokens: int
    cost_usd: float
    latency_ms: int
    raw_metadata: dict

class LLMProvider(ABC):
    @abstractmethod
    def generate(self, prompt: str, model: str, **kwargs) -> LLMResponse:
        pass

    @abstractmethod
    def list_models(self) -> list[str]:
        pass

class FacadeDriver:
    def __init__(self):
        self._providers: dict[str, LLMProvider] = {}
        self._model_to_provider: dict[str, str] = {}

    def register_provider(self, name: str, provider: LLMProvider):
        self._providers[name] = provider
        for model in provider.list_models():
            self._model_to_provider[model] = name

    def generate(self, prompt: str, model: str, **kwargs) -> LLMResponse:
        provider_name = self._model_to_provider.get(model)
        if not provider_name:
            raise ValueError(f"Unknown model: {model}")
        provider = self._providers[provider_name]
        return provider.generate(prompt, model, **kwargs)

    def list_all_models(self) -> list[str]:
        return list(self._model_to_provider.keys())
```

## Provider implementations

Each provider implements the `LLMProvider` interface:

```python
class AzureOpenAIProvider(LLMProvider):
    def __init__(self, endpoint: str, api_key: str):
        self._client = AzureOpenAI(endpoint=endpoint, api_key=api_key)

    def generate(self, prompt: str, model: str, **kwargs) -> LLMResponse:
        start = time.time()
        response = self._client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": prompt}],
            **kwargs
        )
        latency = int((time.time() - start) * 1000)
        return LLMResponse(
            content=response.choices[0].message.content,
            model=model,
            provider="azure_openai",
            input_tokens=response.usage.prompt_tokens,
            output_tokens=response.usage.completion_tokens,
            cost_usd=self._calculate_cost(model, response.usage),
            latency_ms=latency,
            raw_metadata=response.model_dump()
        )

    def list_models(self) -> list[str]:
        return ["gpt-4o", "gpt-4.1", "o1", "o3"]

    def _calculate_cost(self, model: str, usage) -> float:
        pricing = {
            "gpt-4o": {"input": 2.50, "output": 10.00},
            "gpt-4.1": {"input": 2.00, "output": 8.00},
            "o1": {"input": 15.00, "output": 60.00},
            "o3": {"input": 10.00, "output": 40.00},
        }
        rates = pricing.get(model, {"input": 0, "output": 0})
        return (usage.prompt_tokens * rates["input"]
                + usage.completion_tokens * rates["output"]) / 1_000_000
```

## Routing logic

The FacadeDriver can route requests based on:

### Cost optimization

```python
def generate_cheapest(self, prompt: str, **kwargs) -> LLMResponse:
    """Route to the cheapest available model."""
    models_by_cost = sorted(
        self.list_all_models(),
        key=lambda m: self._get_pricing(m)["input"]
    )
    for model in models_by_cost:
        try:
            return self.generate(prompt, model, **kwargs)
        except Exception:
            continue
    raise RuntimeError("All models failed")
```

### Latency optimization

```python
def generate_fastest(self, prompt: str, **kwargs) -> LLMResponse:
    """Route to the fastest available model."""
    # Track rolling average latency per model
    fastest = min(self._latency_tracker.models(), key=lambda m: self._latency_tracker.avg(m))
    return self.generate(prompt, fastest, **kwargs)
```

### Capability-based routing

```python
def generate_for_task(self, prompt: str, task: str, **kwargs) -> LLMResponse:
    """Route based on task requirements."""
    task_models = {
        "simple_qa": "gpt-4o-mini",
        "complex_reasoning": "o3",
        "long_context": "gemini-2.5-pro",
        "code_generation": "claude-4.5-sonnet",
        "cost_sensitive": "qwen-2.5-72b",
    }
    model = task_models.get(task, "gpt-4o")
    return self.generate(prompt, model, **kwargs)
```

### Fallback chains

```python
def generate_with_fallback(self, prompt: str, models: list[str], **kwargs) -> LLMResponse:
    """Try models in order, fall back on failure."""
    for model in models:
        try:
            return self.generate(prompt, model, **kwargs)
        except Exception as e:
            logger.warning(f"Model {model} failed: {e}")
            continue
    raise RuntimeError(f"All {len(models)} models failed")
```

## Cost tracking

The FacadeDriver tracks costs per provider, per model, per request. This is where the
LangChain PR #39351 fix becomes relevant: Perplexity costs were underreported by 10x because
`num_search_queries` was not propagated to `UsageMetadata.output_token_details`. The
FacadeDriver's cost tracking handles this:

```python
class CostTracker:
    def __init__(self):
        self._costs: dict[str, float] = {}
        self._token_counts: dict[str, dict] = {}

    def record(self, response: LLMResponse):
        key = f"{response.provider}/{response.model}"
        self._costs[key] = self._costs.get(key, 0) + response.cost_usd
        self._token_counts.setdefault(key, {"input": 0, "output": 0})
        self._token_counts[key]["input"] += response.input_tokens
        self._token_counts[key]["output"] += response.output_tokens

    def report(self) -> dict:
        return {
            "total_cost": sum(self._costs.values()),
            "by_provider_model": {
                k: {
                    "cost": v,
                    "tokens": self._token_counts[k]
                }
                for k, v in self._costs.items()
            }
        }
```

## The 30+ models behind FacadeDriver at BPI VA

At Airbnb BPI Virtual Analyst, the FacadeDriver orchestrates 30+ model integrations:

| Provider | Models | Use Case |
|----------|--------|----------|
| Azure OpenAI | GPT-4o, GPT-4.1, o1, o3 | General QA, complex reasoning |
| AWS Bedrock | Claude 3.5 Sonnet, Claude 4.5 Sonnet, Haiku | Code generation, long context |
| Google Vertex | Gemini 2.0 Flash, Gemini 2.5 Pro | Long context, multimodal |
| vLLM | Qwen 2.5 72B, Llama 3.3 70B, Mixtral 8x22B | Cost-sensitive, self-hosted |
| Perplexity | sonar-reasoning-pro | Web-grounded QA |

## Why this matters for eval

The FacadeDriver is not just a production abstraction. It is also the foundation of the eval
harness. When you run 23+ agent versions against 1,690 ground-truth samples, each agent
version may use a different model configuration. The FacadeDriver lets you:

1. Run the same eval harness against any model configuration
2. Compare costs across model configurations
3. Detect when a model swap causes regressions (via flip-analysis, Chapter 4)
4. Test new models before adding them to production

Without the FacadeDriver, every model swap requires code changes in the eval harness. With
it, you just register a new model and run the eval.

## Common mistakes

1. **No unified response format**: Each provider returns a different response shape. Without
   a unified `LLMResponse`, your application code is full of provider-specific parsing.
2. **No cost tracking**: Costs are an afterthought. You find out at the end of the month
   that a model swap tripled your bill.
3. **No fallback**: A single provider outage takes down your entire system.
4. **No streaming support**: The FacadeDriver only supports synchronous calls. Streaming
   requires a separate code path, which means provider-specific code leaks.
5. **Tight coupling to one SDK**: Building directly on LangChain or litellm without a
   FacadeDriver layer. When the SDK changes, every call site breaks.

## Summary

The FacadeDriver pattern is the single most important abstraction in a multi-LLM production
system. It enables cost optimization, latency optimization, fallback chains, and unified
eval across model configurations. The 30+ model FacadeDriver at BPI VA has saved hundreds
of engineering hours and made the LLM infrastructure portable across cloud providers.

---

Previous: [Chapter 2: Ground Truth](02-ground-truth.md)
Next: Chapter 4: Dual-Model A/B and Flip Analysis (planned)
