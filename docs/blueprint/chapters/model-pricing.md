# Model Pricing & Cost Calculation

## Why this matters

Every API call costs money. A harness that doesn't track costs will surprise users with unexpected bills. The pricing data below is extracted from the actual Claude Code source (`modelCost.ts`).

## Anthropic Model Pricing (per million tokens)

| Model | Input | Output | Cache Write | Cache Read | Web Search |
|-------|-------|--------|-------------|------------|------------|
| Haiku 3.5 | $0.80 | $4.00 | $1.00 | $0.08 | $0.01/req |
| Haiku 4.5 | $1.00 | $5.00 | $1.25 | $0.10 | $0.01/req |
| Sonnet 3.5v2 | $3.00 | $15.00 | $3.75 | $0.30 | $0.01/req |
| Sonnet 3.7 | $3.00 | $15.00 | $3.75 | $0.30 | $0.01/req |
| Sonnet 4 | $3.00 | $15.00 | $3.75 | $0.30 | $0.01/req |
| Sonnet 4.5 | $3.00 | $15.00 | $3.75 | $0.30 | $0.01/req |
| Sonnet 4.6 | $3.00 | $15.00 | $3.75 | $0.30 | $0.01/req |
| Opus 4 | $15.00 | $75.00 | $18.75 | $1.50 | $0.01/req |
| Opus 4.1 | $15.00 | $75.00 | $18.75 | $1.50 | $0.01/req |
| Opus 4.5 | $5.00 | $25.00 | $6.25 | $0.50 | $0.01/req |
| Opus 4.6 | $5.00 | $25.00 | $6.25 | $0.50 | $0.01/req |
| **Opus 4.6 Fast** | **$30.00** | **$150.00** | **$37.50** | **$3.00** | $0.01/req |

!!! warning "Fast mode pricing"
    Fast mode (Opus 4.6 only) costs **6x normal** for priority processing. Cache write is 1.25x input, cache read is 0.1x input.

## Cache pricing formula

```
Cache write  = input_price * 1.25
Cache read   = input_price * 0.10
```

Cache reads are **10x cheaper** than regular input. This is why prompt cache sharing (`CacheSafeParams`) is so important for multi-agent systems.

## Python cost calculator

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class ModelPricing:
    """Pricing per million tokens."""

    input_tokens: float
    output_tokens: float
    cache_write_tokens: float
    cache_read_tokens: float
    web_search_per_request: float = 0.01


# Extracted from Claude Code source (modelCost.ts)
PRICING: dict[str, ModelPricing] = {
    "claude-haiku-3-5": ModelPricing(0.80, 4.00, 1.00, 0.08),
    "claude-haiku-4-5": ModelPricing(1.00, 5.00, 1.25, 0.10),
    "claude-sonnet-4-6": ModelPricing(3.00, 15.00, 3.75, 0.30),
    "claude-opus-4-6": ModelPricing(5.00, 25.00, 6.25, 0.50),
    "claude-opus-4-6-fast": ModelPricing(30.00, 150.00, 37.50, 3.00),
    # OpenAI models (approximate public pricing)
    "gpt-4.1": ModelPricing(2.00, 8.00, 0.50, 0.50),
    "gpt-4.1-mini": ModelPricing(0.40, 1.60, 0.10, 0.10),
    "gpt-4.1-nano": ModelPricing(0.10, 0.40, 0.025, 0.025),
    "o3": ModelPricing(2.00, 8.00, 0.50, 0.50),
    "o4-mini": ModelPricing(1.10, 4.40, 0.275, 0.275),
}


def calculate_cost(
    model: str,
    input_tokens: int,
    output_tokens: int,
    cache_creation_tokens: int = 0,
    cache_read_tokens: int = 0,
    web_search_requests: int = 0,
) -> float:
    """Calculate USD cost for an API call."""
    pricing = PRICING.get(model)
    if not pricing:
        return 0.0

    cost = (
        (input_tokens / 1_000_000) * pricing.input_tokens
        + (output_tokens / 1_000_000) * pricing.output_tokens
        + (cache_creation_tokens / 1_000_000) * pricing.cache_write_tokens
        + (cache_read_tokens / 1_000_000) * pricing.cache_read_tokens
        + web_search_requests * pricing.web_search_per_request
    )
    return round(cost, 6)
```

## Cost tracker (per-session)

```python
from dataclasses import dataclass, field


@dataclass
class CostTracker:
    total_cost_usd: float = 0.0
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    total_cache_read_tokens: int = 0
    total_cache_creation_tokens: int = 0
    total_web_search_requests: int = 0
    model_usage: dict[str, dict] = field(default_factory=dict)

    def record(self, model: str, usage: dict) -> float:
        """Record usage from an API response. Returns cost of this call."""
        input_tok = usage.get("input_tokens", 0)
        output_tok = usage.get("output_tokens", 0)
        cache_read = usage.get("cache_read_input_tokens", 0)
        cache_create = usage.get("cache_creation_input_tokens", 0)

        cost = calculate_cost(model, input_tok, output_tok, cache_create, cache_read)

        self.total_cost_usd += cost
        self.total_input_tokens += input_tok
        self.total_output_tokens += output_tok
        self.total_cache_read_tokens += cache_read
        self.total_cache_creation_tokens += cache_create

        if model not in self.model_usage:
            self.model_usage[model] = {"input": 0, "output": 0, "cost": 0.0}
        self.model_usage[model]["input"] += input_tok
        self.model_usage[model]["output"] += output_tok
        self.model_usage[model]["cost"] += cost

        return cost
```

## Budget enforcement

```python
@dataclass(frozen=True)
class BudgetPolicy:
    max_cost_usd: float = 5.0
    max_tokens: int = 500_000
    warn_at_pct: float = 0.8  # warn at 80%


def check_budget(tracker: CostTracker, policy: BudgetPolicy) -> str | None:
    """Return an error message if budget is exceeded, None otherwise."""
    if tracker.total_cost_usd >= policy.max_cost_usd:
        return f"Budget exceeded: ${tracker.total_cost_usd:.4f} >= ${policy.max_cost_usd}"
    total_tokens = tracker.total_input_tokens + tracker.total_output_tokens
    if total_tokens >= policy.max_tokens:
        return f"Token limit exceeded: {total_tokens:,} >= {policy.max_tokens:,}"
    return None
```

## Using with both SDKs

=== "Anthropic"

    ```python
    response = client.messages.create(...)
    cost = tracker.record(response.model, {
        "input_tokens": response.usage.input_tokens,
        "output_tokens": response.usage.output_tokens,
        "cache_read_input_tokens": getattr(
            response.usage, "cache_read_input_tokens", 0
        ),
        "cache_creation_input_tokens": getattr(
            response.usage, "cache_creation_input_tokens", 0
        ),
    })
    print(f"This call: ${cost:.4f} | Session total: ${tracker.total_cost_usd:.4f}")
    ```

=== "OpenAI"

    ```python
    response = client.responses.create(...)
    cost = tracker.record(response.model, {
        "input_tokens": response.usage.input_tokens,
        "output_tokens": response.usage.output_tokens,
    })
    print(f"This call: ${cost:.4f} | Session total: ${tracker.total_cost_usd:.4f}")
    ```

## Build it yourself

1. Create a pricing table for every model you support
2. Track usage per-call and per-session with a `CostTracker`
3. Enforce budgets at the query loop level (check after each turn)
4. Show cost to the user after each turn (transparency builds trust)
