---
id: oauth
title: OAuth & Authentication
sidebar_position: 1
description: OAuth 2.0 and API authentication for BlackRoad OS — all requests must carry a BlackRoad-issued bearer token obtained through the converter API.
---

# OAuth & Authentication

> **Status:** Production  
> **Owner:** @blackboxprogramming

BlackRoad OS uses **OAuth 2.0 with PKCE** for user-facing flows and **short-lived bearer tokens** for service-to-service authentication. Access is mediated by the BlackRoad Converter API — contributors and integrations must obtain a token through this gateway before reaching any internal service.

---

## Token Flow

```
Client / Contributor
        │
        │  1. Request access via Converter API
        ▼
  converter.blackroad.io/token
        │
        │  2. Returns short-lived bearer token (br_*)
        ▼
  api.blackroad.io/v1/*   (all requests carry Authorization: Bearer br_*)
```

---

## Obtaining a Token

### Converter API — Token Exchange

```bash
POST https://converter.blackroad.io/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&scope=intent:read intent:write policy:evaluate
```

**Response:**

```json
{
  "access_token": "br_user_xxxxxxxxxxxxxxxxxxxx",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "intent:read intent:write policy:evaluate"
}
```

Use the returned `access_token` as the `Authorization: Bearer` header on all subsequent API requests.

---

## OAuth 2.0 PKCE Flow (User Applications)

For browser-based or mobile clients:

### Step 1 — Generate a code verifier and challenge

```typescript
import crypto from 'crypto';

const codeVerifier = crypto.randomBytes(32).toString('base64url');
const codeChallenge = crypto
  .createHash('sha256')
  .update(codeVerifier)
  .digest('base64url');
```

### Step 2 — Redirect user to the authorization endpoint

```
GET https://auth.blackroad.io/oauth/authorize
  ?response_type=code
  &client_id=YOUR_CLIENT_ID
  &redirect_uri=https://yourapp.example.com/callback
  &scope=openid profile intent:read
  &code_challenge=CODE_CHALLENGE
  &code_challenge_method=S256
  &state=RANDOM_STATE
```

### Step 3 — Exchange the code for tokens

```bash
POST https://auth.blackroad.io/oauth/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "code": "AUTHORIZATION_CODE",
  "redirect_uri": "https://yourapp.example.com/callback",
  "client_id": "YOUR_CLIENT_ID",
  "code_verifier": "CODE_VERIFIER"
}
```

---

## Token Types

| Type | Prefix | Lifetime | Use Case |
|------|--------|----------|----------|
| User | `br_user_` | 1 hour | End-user API access |
| Agent | `br_agent_` | 24 hours | Autonomous agent calls |
| System | `br_system_` | 7 days | Internal service mesh |
| Test | `br_test_` | 1 hour | Development / staging |

---

## Scopes

| Scope | Description |
|-------|-------------|
| `intent:read` | Read intents |
| `intent:write` | Create and update intents |
| `policy:evaluate` | Evaluate governance policies |
| `ledger:read` | Read audit trail |
| `ledger:write` | Append audit events |
| `agent:read` | Read agent registry |
| `agent:register` | Register new agents |
| `admin` | Full administrative access (restricted) |

---

## Contributor Access — Converter API Requirement

> **All new contributors must obtain a Converter API token before accessing any BlackRoad infrastructure.**

1. Contact **@blackboxprogramming** or **@lucidia** to request a contributor `client_id` and `client_secret`.
2. Exchange credentials at `converter.blackroad.io/token`.
3. Include the resulting bearer token in all API requests and CI/CD pipelines.
4. Tokens are scoped to the minimum permissions required. Requests for broader scope require explicit approval.

Contributors who do not have a valid Converter API token will receive `401 Unauthorized` on all internal endpoints.

---

## Token Verification (Server-Side)

```typescript
import jwt from 'jsonwebtoken';

function verifyToken(authHeader: string): TokenPayload {
  const token = authHeader?.replace('Bearer ', '');
  if (!token) throw new Error('Missing token');

  return jwt.verify(token, process.env.BLACKROAD_JWT_PUBLIC_KEY!, {
    algorithms: ['RS256'],
    issuer: 'https://auth.blackroad.io',
    audience: 'https://api.blackroad.io',
  }) as TokenPayload;
}
```

---

## Revoking Tokens

```bash
POST https://auth.blackroad.io/oauth/revoke
Authorization: Bearer br_system_xxxx
Content-Type: application/json

{
  "token": "br_user_xxxxxxxxxxxxxxxxxxxx",
  "token_type_hint": "access_token"
}
```

---

## Environment Variables

```bash
BLACKROAD_CLIENT_ID=your_client_id
BLACKROAD_CLIENT_SECRET=your_client_secret   # never commit
BLACKROAD_JWT_PUBLIC_KEY=-----BEGIN PUBLIC KEY-----...
AUTH_ISSUER=https://auth.blackroad.io
CONVERTER_API_URL=https://converter.blackroad.io
```

---

## References

- [API Design](./api-design.md)
- [Vendor API Gateway](./vendor-api-gateway.md)
- [Stripe Integration](./stripe.md)
- [CONTRIBUTING](../../CONTRIBUTING.md)
