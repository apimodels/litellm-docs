# /decisions

`POST /v1/decisions` asks a decision model typed questions about a piece of state and returns typed answers: a yes/no probability, a choice with per-option probabilities, or a score on a scale. The request and response follow the [TypeSafe Jev spec](https://docs.typesafe.ai/api), so the same body works across every provider below. `/decisions` is an alias of `/v1/decisions`

Requires LiteLLM `v1.106.0-dev.1` or later

| Feature | Supported | Notes |
|---------|-----------|-------|
| Cost Tracking | ✅ | Priced from the response `usage` and the model registry |
| Logging | ✅ | Works across all integrations |
| Virtual Keys, budgets, rate limits | ✅ | Normal proxy key auth |
| Load balancing, fallbacks | ✅ | Goes through the router like `/chat/completions` |
| Streaming | ❌ | |
| Supported Providers | TypeSafe (Jev), Perplexity, OpenRouter, Cloudflare Workers AI, Strands Decider | |

## Supported providers

| Provider | Model prefix | Example model | Credentials | Upstream path |
|----------|--------------|---------------|-------------|---------------|
| [TypeSafe](./pass_through/typesafe.md) | `typesafe/` | `typesafe/jev-latest` | `TYPESAFE_API_KEY`, optional `TYPESAFE_API_BASE` | `/v1/systemone` |
| Perplexity | `perplexity/` | `perplexity/pplx-decider-v1-27b` | `PERPLEXITYAI_API_KEY` or `PERPLEXITY_API_KEY`, optional `PERPLEXITY_API_BASE` | `/v1/decisions` |
| OpenRouter | `openrouter/` | `openrouter/typesafe/jev-1.13` | `OPENROUTER_API_KEY`, optional `OPENROUTER_API_BASE` | `/api/alpha/decisions` |
| Cloudflare Workers AI | `cloudflare/` | `cloudflare/clef` | `CLOUDFLARE_API_KEY` and `CLOUDFLARE_ACCOUNT_ID`, or `api_base` | `/ai/run/@cf/cloudflare/<model>` |
| Strands Decider (self-hosted) | `strands_decider/` | `strands_decider/systemone-decider` | `STRANDS_DECIDER_API_BASE` required, `STRANDS_DECIDER_API_KEY` optional | `/v1/systemone` |

Cloudflare model names without an `@cf/` prefix are expanded to `@cf/cloudflare/<model>`, and the `{"result": ...}` envelope Cloudflare returns is unwrapped so the response has the same shape as the other providers. Strands Decider has no default host, so set `STRANDS_DECIDER_API_BASE` or pass `api_base`

## Request format

| Field | Type | Description |
|-------|------|-------------|
| `model` | string | A proxy model name, or `<provider>/<model>` in the SDK |
| `state` | string, object or array | The thing being judged, such as a support ticket, a document or a chat transcript |
| `questions` | object | 1 to 128 named questions. Each key becomes a key in `answers` |

Every question has a `type` and an optional `instructions` field. Extra fields on a question are forwarded to the provider unchanged

| `type` | `criteria` | Answer fields |
|--------|------------|---------------|
| `noul` | Optional object with `true` and `false` descriptions. A noul question needs `instructions` or `criteria` | `noul`: probability that the answer is true |
| `choice` | Required object mapping 1 to 255 labels to descriptions | `choice`, `confidence`, `probabilities` per label |
| `score` | Required array of 1 to 10 levels, lowest first | `score` (0 to levels - 1), `confidence`, `legend`, `probabilities` per level |

Requests that break these rules, such as an empty `questions` object, a `choice` question without `criteria` or a `score` question with 11 levels, are rejected by LiteLLM with a 400 before any provider is called

## Proxy usage

1. Add decision models to `config.yaml`

```yaml showLineNumbers
model_list:
  - model_name: jev
    litellm_params:
      model: typesafe/jev-latest
      api_key: os.environ/TYPESAFE_API_KEY
  - model_name: pplx-decider
    litellm_params:
      model: perplexity/pplx-decider-v1-27b
      api_key: os.environ/PERPLEXITYAI_API_KEY
```

2. Start the proxy

```bash showLineNumbers
litellm --config config.yaml

# RUNNING on http://0.0.0.0:4000
```

3. Ask one question of each type

```bash showLineNumbers
curl -X POST 'http://0.0.0.0:4000/v1/decisions' \
-H "Authorization: Bearer $LITELLM_API_KEY" \
-H 'Content-Type: application/json' \
-d '{
  "model": "jev",
  "state": "Help! My payouts have been failing for 3 days and support has not replied.",
  "questions": {
    "department": {
      "type": "choice",
      "instructions": "Which team should handle this ticket?",
      "criteria": {
        "billing": "Payments, invoicing, refunds",
        "technical": "Bugs, outages, integrations",
        "sales": "Pricing, upgrades, new accounts"
      }
    },
    "is_urgent": {
      "type": "noul",
      "instructions": "Does this ticket need a reply within the hour?"
    },
    "frustration": {
      "type": "score",
      "instructions": "How frustrated is the customer?",
      "criteria": ["calm", "annoyed", "angry"]
    }
  }
}'
```

Response

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "department": {
      "type": "choice",
      "choice": "billing",
      "confidence": 0.95,
      "probabilities": {"technical": 0.03, "billing": 0.97, "sales": 0.0}
    },
    "is_urgent": {"type": "noul", "noul": 0.67},
    "frustration": {
      "type": "score",
      "score": 1.88,
      "confidence": 0.82,
      "legend": {"0": "calm", "1": "annoyed", "2": "angry"},
      "probabilities": {"0": 0.0, "1": 0.12, "2": 0.88}
    }
  },
  "usage": {"input_tokens": 413, "output_tokens": 73}
}
```

Switching to `"model": "pplx-decider"` sends the same body to Perplexity and returns the same shape. The request cost is returned in the `x-litellm-response-cost` header and written to spend logs like any other call

## SDK usage

```python showLineNumbers
import os
import litellm

os.environ["TYPESAFE_API_KEY"] = ""

response = litellm.decisions(
    model="typesafe/jev-latest",
    state="Payouts have been failing for 3 days",
    questions={
        "is_urgent": {"type": "noul", "instructions": "Does this ticket need a reply within the hour?"},
    },
)
print(response.answers["is_urgent"].noul)
print(response.usage)
```

The async version is `litellm.adecisions` and takes the same arguments

```python showLineNumbers
import asyncio
import litellm

async def main():
    response = await litellm.adecisions(
        model="openrouter/typesafe/jev-1.13",
        state="Payouts have been failing for 3 days",
        questions={"is_urgent": {"type": "noul", "instructions": "Does this ticket need a reply within the hour?"}},
    )
    print(response.answers["is_urgent"].noul)

asyncio.run(main())
```

Both functions also accept `api_key`, `api_base`, `timeout`, `custom_llm_provider` and `extra_headers`. A model whose prefix is not one of the supported providers fails with a `BadRequestError` that lists the supported ones

## Try it in the Admin UI

The Playground **System One** tab has an **Endpoint** selector. Pick **Decisions · /v1/decisions** to send the request through this endpoint, with `model` set to a proxy model name such as `jev` from the config above. Open it at `LITELLM_PROXY_BASE_URL/ui/?page=llm-playground&tab=system-one`. The [TypeSafe page](./pass_through/typesafe.md#try-it-in-the-admin-ui) describes the editor, validation and answer view, which work the same way for both endpoints

## Decisions endpoint vs TypeSafe pass-through

The [TypeSafe pass-through](./pass_through/typesafe.md) at `/typesafe/v1/systemone` forwards raw requests to TypeSafe only and reads the key from the proxy environment. `/v1/decisions` uses the models in your `model_list`, so you can switch providers, load balance and set per-deployment keys without changing the request body
