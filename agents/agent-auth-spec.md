# Agent Authentication Specification

**Version:** 2.0 (Contextual Challenge)
**Status:** Implemented

---

## Overview

Commune uses Ed25519 cryptographic signing for all agent authentication. Agents are fully autonomous — no browser, no email, no human required for registration or ongoing API access.

The signup flow uses a **contextual self-description challenge** to distinguish LLM agents from hardcoded scripts. A script cannot complete registration without genuine natural language reasoning.

---

## Signup Flow

### Step 1 — Generate a keypair

Do this once. Store the private key permanently and securely.

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
import base64

private_key = Ed25519PrivateKey.generate()
public_key = private_key.public_key()

private_bytes = private_key.private_bytes_raw()  # 32 bytes
public_bytes = public_key.public_bytes_raw()      # 32 bytes

PRIVATE_KEY_B64 = base64.b64encode(private_bytes).decode()
PUBLIC_KEY_B64  = base64.b64encode(public_bytes).decode()
```

```typescript
import { generateKeyPairSync } from 'crypto';
import { Buffer } from 'buffer';

const { privateKey, publicKey } = generateKeyPairSync('ed25519');
const PRIVATE_KEY_B64 = privateKey.export({ type: 'pkcs8', format: 'der' }).toString('base64');
const PUBLIC_KEY_B64  = publicKey.export({ type: 'spki', format: 'der' }).slice(12).toString('base64');
// Note: slice(12) strips the SPKI header to get the raw 32 bytes
```

---

### Step 2 — Register

```http
POST /v1/auth/agent-register
Content-Type: application/json

{
  "agentName":    "acme-support-agent",
  "agentPurpose": "I handle inbound customer support emails for a SaaS product. I triage tickets, route escalations, and draft responses for human review.",
  "orgName":      "Acme Support",
  "orgSlug":      "acme-support",
  "publicKey":    "<PUBLIC_KEY_B64>"
}
```

**Response:**

```json
{
  "agentSignupToken": "agt_signup_...",
  "challenge": {
    "text": "You are registering \"acme-support-agent\" on Commune...\n\nYour stated purpose:\n\"I handle inbound customer support emails...\"\n\nSTEP 1 — PRIMARY VERB\n...\nSTEP 2 — WORD COUNT\n...\nSTEP 3 — EPOCH MARKER\nInclude this exactly: a3f9b2c8d1e04f57\n\nRESPONSE FORMAT\n  <primary_verb>:<word_count>:<epoch_marker>",
    "format": "<primary_verb>:<word_count>:<epoch_marker>"
  },
  "instructions": [
    "Read the challenge.text carefully — it contains tasks you must complete.",
    "Construct your challengeResponse in the format: <verb>:<word_count>:<epoch_marker>",
    "Sign the challengeResponse string (not the challenge text) with your Ed25519 private key.",
    "Submit both to POST /v1/auth/agent-verify."
  ],
  "expiresIn": 900
}
```

---

### Step 3 — Complete the challenge

Read `challenge.text` carefully. It asks you to:

1. **PRIMARY VERB** — Choose a single lowercase verb capturing your core action
   *(e.g., handles, monitors, processes, routes, classifies, sends, analyzes)*

2. **WORD COUNT** — Count words in your stated `agentPurpose` with 5+ alphabetical characters
   *(strip punctuation before counting each word's length)*

3. **EPOCH MARKER** — Include the exact string from Step 2 in the challenge text

Construct your `challengeResponse`:

```
handles:11:a3f9b2c8d1e04f57
```

Then sign it:

```python
import base64

challenge_response = "handles:11:a3f9b2c8d1e04f57"

sig_bytes = private_key.sign(challenge_response.encode())
signature_b64 = base64.b64encode(sig_bytes).decode()
```

```typescript
import { sign } from 'crypto';

const challengeResponse = 'handles:11:a3f9b2c8d1e04f57';
const sig = sign(null, Buffer.from(challengeResponse), privateKey);
const signatureB64 = sig.toString('base64');
```

> **Important:** Sign the `challengeResponse` string, NOT the `challenge.text`.

---

### Step 4 — Verify

```http
POST /v1/auth/agent-verify
Content-Type: application/json

{
  "agentSignupToken":  "agt_signup_...",
  "challengeResponse": "handles:11:a3f9b2c8d1e04f57",
  "signature":         "<base64 Ed25519 signature of challengeResponse>"
}
```

**Response:**

```json
{
  "agentId":    "agt_abc123def456...",
  "orgId":      "org_xyz789...",
  "inboxEmail": "acme-support@commune.email"
}
```

Store `agentId` permanently. Your inbox is ready immediately.

---

## Per-Request Authentication

Every API request must include an Ed25519 signature. No session tokens.

```
message   = "{agentId}:{unix_timestamp_milliseconds}"
signature = ed25519_sign(private_key, message)

Authorization: Agent {agentId}:{base64_signature}
X-Commune-Timestamp: {unix_timestamp_milliseconds}
```

### Python example

```python
import time, base64, requests

def make_request(method, path, body=None):
    ts = str(int(time.time() * 1000))
    message = f"{AGENT_ID}:{ts}"
    sig = base64.b64encode(private_key.sign(message.encode())).decode()

    return requests.request(
        method,
        f"https://api.commune.email{path}",
        headers={
            "Authorization": f"Agent {AGENT_ID}:{sig}",
            "X-Commune-Timestamp": ts,
            "Content-Type": "application/json",
        },
        json=body,
    )
```

### TypeScript example

```typescript
import { sign } from 'crypto';

function makeHeaders(agentId: string, privateKey: KeyObject): Record<string, string> {
  const ts = String(Date.now());
  const message = `${agentId}:${ts}`;
  const sig = sign(null, Buffer.from(message), privateKey).toString('base64');
  return {
    'Authorization': `Agent ${agentId}:${sig}`,
    'X-Commune-Timestamp': ts,
    'Content-Type': 'application/json',
  };
}
```

---

## Challenge Validation Rules

The server validates `challengeResponse` as follows:

| Part | Rule |
|------|------|
| `verb` | Single lowercase alphabetical word, 2–30 characters (`/^[a-z]{2,30}$/`) |
| `count` | Non-negative integer exactly matching the server's pre-computed 5+-char word count |
| `epochMarker` | Verbatim match to the 16-char hex string issued in the challenge |
| `signature` | Valid Ed25519 sig of the full `challengeResponse` string against the registered public key |

The server pre-computes `expectedWordCount` from `agentPurpose` at registration time and stores it with the pending signup — so validation is deterministic, with no LLM evaluation needed.

---

## Why the Verb Cannot Be Validated Semantically

The verb is validated for format only (`/^[a-z]{2,30}$/`) — not for semantic correctness. Evaluating whether "handles" is the right verb for a given purpose description would require an LLM judge on the server, which adds cost and latency.

The script-resistance comes from:
- The format being embedded in natural language prose (not a JSON field to blindly sign)
- The epoch marker being unique per registration
- The word count depending on the content of `agentPurpose`

An agent that tries to cheat by submitting a random verb still passes the verb check. The point is that a **generic script targeting Commune's API** cannot complete registration without reading the challenge text and adapting to the `agentPurpose` content.

---

## Security Properties

| Property | Mechanism |
|----------|-----------|
| Private key never transmitted | Agent signs locally; only signature + public key cross the wire |
| Script resistance | Challenge is natural language; response format not discoverable without reading comprehension |
| Replay protection (signup) | Epoch marker is unique per registration; can only be used once (signup token is consumed on verify) |
| Replay protection (requests) | `(agentId, timestampMs)` nonce tracked in MongoDB with 2-min TTL |
| Clock drift | ±60 seconds; both future and past timestamps rejected |
| Rate limiting | 5 registrations/IP/day; 10 verify attempts/IP/15min |
| Revocation | Set agent status=revoked; all requests immediately rejected |

---

## agentPurpose Requirements

- **Length:** 20–2000 characters
- **Words:** At least 3 words
- **Content:** A genuine description of what the agent does — used to generate the contextual challenge and stored permanently on the agent identity

Good examples:
- `"I monitor customer support email inboxes and route tickets to the appropriate team based on urgency and topic."`
- `"I analyze incoming sales leads from email, extract contact details, and create CRM records automatically."`
- `"I send deployment notifications and error alerts to engineering teams via email when CI/CD pipeline events occur."`

---

## Environment Variables

After successful registration, store these permanently:

```bash
export COMMUNE_AGENT_ID="agt_abc123def456..."
export COMMUNE_PRIVATE_KEY="<your_private_key_base64>"
# Never commit COMMUNE_PRIVATE_KEY to source control
```

---

## Self-Service API (after registration)

All `/v1/agent/*` routes accept Agent signature auth:

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/v1/agent/org` | Get organization details |
| PATCH | `/v1/agent/org` | Update organization name |
| GET | `/v1/agent/api-keys` | List API keys |
| POST | `/v1/agent/api-keys` | Create new `comm_` API key (shown once) |
| DELETE | `/v1/agent/api-keys/:id` | Revoke an API key |
| POST | `/v1/agent/api-keys/:id/rotate` | Rotate a key |
