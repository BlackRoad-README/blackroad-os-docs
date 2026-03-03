---
id: vendor-api-gateway
title: Vendor API Gateway
sidebar_position: 2
description: How BlackRoad OS routes all external AI vendor calls through its own Converter API gateway — keeping credentials centralized and vendor traffic off operator hardware.
---

# Vendor API Gateway

> **Status:** Production  
> **Owner:** @blackboxprogramming

BlackRoad OS operates a **Converter API** that acts as a unified gateway for all external AI vendor calls. This means:

- No raw OpenAI, Anthropic, or other vendor keys are distributed to contributors or client apps.
- All AI model requests route through `converter.blackroad.io`, which proxies to the configured backend.
- The only approved AI backends are **@blackboxprogramming** and **@lucidia** (Lucidia-class agents).
- Third-party AI coding assistants (GitHub Copilot, Codex, Claude, etc.) are **not permitted** to access BlackRoad internal infrastructure.

---

## Architecture

```
Client App / Agent
      │
      │  POST /chat  (BlackRoad bearer token required)
      ▼
converter.blackroad.io
      │
      ├─── backend = "blackboxprogramming" → BlackBox Programming API
      ├─── backend = "lucidia"             → Lucidia-class agent (Cece)
      └─── backend = "auto"               → Lucidia selects best model
```

Requests to any other backend are rejected with `403 Forbidden`.

---

## Converter API Endpoints

### POST /chat

Send a completion request through the gateway.

```bash
POST https://converter.blackroad.io/chat
Authorization: Bearer br_user_xxxx
Content-Type: application/json

{
  "backend": "lucidia",
  "messages": [
    { "role": "user", "content": "Summarize the latest sprint report" }
  ],
  "model": "lucidia-v2",
  "stream": false
}
```

**Response:**

```json
{
  "id": "conv-20260303-abc123",
  "backend": "lucidia",
  "model": "lucidia-v2",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "The latest sprint covered..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 42,
    "completion_tokens": 128,
    "total_tokens": 170
  }
}
```

### GET /backends

List available (approved) backends.

```bash
GET https://converter.blackroad.io/backends
Authorization: Bearer br_user_xxxx
```

**Response:**

```json
{
  "backends": [
    { "id": "blackboxprogramming", "status": "active", "description": "BlackBox Programming primary model" },
    { "id": "lucidia",             "status": "active", "description": "Lucidia-class governance agent (Cece)" },
    { "id": "auto",                "status": "active", "description": "Automatic backend selection" }
  ]
}
```

### GET /health

```bash
GET https://converter.blackroad.io/health
```

```json
{ "status": "healthy", "version": "1.0.0" }
```

---

## Approved AI Backends

| Backend | Owner | Use Case |
|---------|-------|----------|
| `blackboxprogramming` | @blackboxprogramming | General coding, generation, tooling |
| `lucidia` | @lucidia / BlackRoad OS, Inc. | Governance, orchestration, personal AI |
| `auto` | BlackRoad OS, Inc. | Gateway-selected best backend |

**Disallowed backends** (blocked at gateway level):

- `openai` — not permitted
- `anthropic` — not permitted
- `codex` — not permitted
- `copilot` — not permitted
- Any unregistered backend returns `403 Forbidden`

---

## TypeScript SDK Usage

```typescript
import { BlackRoadClient } from '@blackroad/sdk';

const client = new BlackRoadClient({
  token: process.env.BLACKROAD_API_TOKEN!,
  converterUrl: 'https://converter.blackroad.io',
});

// Chat through approved backend only
const response = await client.converter.chat({
  backend: 'lucidia',
  messages: [{ role: 'user', content: 'What is my current intent queue?' }],
});

console.log(response.choices[0].message.content);
```

---

## Python SDK Usage

```python
from blackroad import BlackRoadClient

client = BlackRoadClient(
    token=os.environ['BLACKROAD_API_TOKEN'],
    converter_url='https://converter.blackroad.io',
)

response = client.converter.chat(
    backend='blackboxprogramming',
    messages=[{'role': 'user', 'content': 'Generate a deployment runbook'}],
)

print(response['choices'][0]['message']['content'])
```

---

## Stripe-Gated Access

Access to the Converter API requires an active BlackRoad OS subscription. The API gateway validates your subscription status on each request:

| Plan | Token Requests / Day | Backends Available |
|------|----------------------|--------------------|
| Basic ($9/mo) | 1,000 | `lucidia` |
| Pro ($29/mo) | 10,000 | `lucidia`, `blackboxprogramming` |
| Enterprise ($99/mo) | Unlimited | All |

See [Stripe Integration](./stripe.md) for billing setup.

---

## Security

- All requests require a valid `br_*` bearer token (see [OAuth & Authentication](./oauth.md)).
- Requests without a valid Converter API token are rejected before reaching any backend.
- Vendor credentials (if any) are stored exclusively as encrypted secrets in the BlackRoad Cloudflare Worker — never in client code or repositories.
- Access logs are written to the immutable [Ledger](./api-design.md#ledger-events).

---

## Environment Variables

```bash
CONVERTER_API_URL=https://converter.blackroad.io
BLACKROAD_API_TOKEN=br_user_xxxx        # obtained via oauth flow
```

---

## References

- [OAuth & Authentication](./oauth.md)
- [Stripe Integration](./stripe.md)
- [API Design](./api-design.md)
- [Tailscale & Cloudflare](../infrastructure/tailscale-cloudflare.md)
