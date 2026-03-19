# Agent Verification Research: Proving "Agent-ness" at Signup

**Status:** Research → Approach 2 selected for implementation
**Last updated:** 2026-02-21

---

## The Problem

Commune's agent authentication uses Ed25519 keypair signing for per-request auth — solid. But signup (registration) has no proof that the registrant is actually an LLM agent. Any script with `crypto` can:

1. Generate an Ed25519 keypair
2. POST `/v1/auth/agent-register` with `{ agentName, orgName, orgSlug, publicKey }`
3. Sign the returned `challenge` nonce
4. POST `/v1/auth/agent-verify` with `{ agentSignupToken, signature }`
5. Get a valid agent identity

We want to raise the bar so that **LLM agents can register easily**, but **hardcoded scripts targeting our API cannot**.

---

## The Core Insight

This is an inverse CAPTCHA — instead of proving you're human, prove you're a reasoning machine.

| Capability | Script | LLM Agent |
|-----------|--------|-----------|
| Generate keypairs | ✓ | ✓ |
| Sign arbitrary bytes | ✓ | ✓ |
| Make HTTP requests | ✓ | ✓ |
| Parse natural language instructions | ✗ | ✓ |
| Reason about open-ended problems | ✗ | ✓ |
| Adapt to context-dependent challenges | ✗ | ✓ |
| Demonstrate semantic understanding | ✗ | ✓ |

---

## Approaches Explored

### Approach 1 — Hidden Instructions Challenge

Server returns a natural language paragraph with the real challenge embedded in prose — not in a structured JSON field. A script expecting `challenge: "chal_abc123"` to sign directly would fail.

Example:
> "Count the vowels in 'commune email platform', multiply by 3, then append the word completing this analogy: inbox is to email as _______ is to voice. Concatenate and sign."

**Pros:** Server validates deterministically (it generated the puzzle), no LLM judge needed.
**Cons:** A sophisticated script could still parse the template if it knows the format.

---

### Approach 2 — Contextual Self-Description Challenge ⭐ IMPLEMENTED

Agent registers with `agentName`, `agentPurpose` (a description of what the agent does), and `publicKey`. Server generates a challenge **specific to the agent's own stated purpose**.

The challenge asks the agent to:
1. **Identify** the primary verb capturing their core action (reasoning about their own purpose)
2. **Count** words in their stated purpose that have 5+ alphabetical characters (requires reading comprehension + arithmetic)
3. **Include** a random epoch marker (prevents replay/caching)

Construct `challengeResponse = "<verb>:<count>:<epochMarker>"` and sign it.

**Why this works:**
- The format is embedded in natural language prose — not a structured JSON field to blindly sign
- The epoch marker changes every registration (no cached solutions)
- The verb requires reading comprehension of the agent's own purpose
- The word count requires processing the stated purpose text
- Server validates deterministically: it pre-computes expected word count + stores the epoch marker

**Why scripts fail:**
- Scripts hardcoded for Commune's old API expect to sign `challenge: "chal_xxx"` directly
- The new format requires parsing natural language to find the required construction format
- The verb cannot be hardcoded — it must semantically match the stated purpose
- Wrong word count → rejected before signature check

**See implementation:** `backend/src/services/agentIdentityService.ts`

---

### Approach 3 — Model Attestation (Gold Standard) ⭐ HIGHLIGHT

> **This is the strongest long-term approach and should be a Commune-defining feature.**

When an LLM makes an API call, the AI provider (Anthropic, OpenAI, Google) can cryptographically prove that actual inference happened. The agent requests its model to sign a registration statement; Commune verifies against the provider's public key.

```
Agent → Anthropic API: "Generate a Commune registration statement"
Anthropic → Agent: {
  response: "I am registering as X...",
  attestation: {
    modelId: "claude-opus-4-6",
    inferenceId: "inf_abc123",
    timestamp: 1740000000,
    signature: "<Anthropic's Ed25519 private key signature>"
  }
}

Agent → Commune /v1/auth/agent-register: {
  agentName: "...",
  publicKey: "...",
  attestation: { modelId, inferenceId, timestamp, signature }
}

Commune → Anthropic's public key: verify(attestation) ✓
```

**Why this is the gold standard:**
- **Cryptographically proves** an actual LLM inference happened — cannot be faked by any script
- Ties identity to a specific model and provider
- Providers already have the infrastructure (usage receipts, audit logs)
- Enables tier-based trust: attested agents get higher rate limits, SLA guarantees

**Immediate partial version:**
- Agent includes their `ANTHROPIC_API_KEY` during signup
- Commune makes a test inference call to verify it's a real, active, paying account
- Not as strong as full attestation but proves there's a real LLM customer behind the agent

**Long-term:**
- Partner with Anthropic, OpenAI, Google to expose a `/v1/attestation` endpoint
- Agents can request a signed receipt for any inference
- This receipt is the "passport" that grants full trust at any service implementing the standard
- Commune could be the first platform to support and require this

**This would be industry-defining** — analogous to domain-validated TLS certs, but for AI agent identity.

---

### Approach 4 — Multi-Turn Contextual Conversation

Registration becomes a 3-round dialogue. Each round's question depends on the agent's previous answer — context the server injects from prior responses.

- Round 1: "What is your purpose?"
- Round 2: Server pulls a word from their Round 1 answer: "You mentioned 'support'. What is the primary failure mode of an agent doing support?"
- Round 3: "How does your Round 2 answer relate to email latency?"

A script cannot anticipate Round 2/3 without effectively being an LLM.

**Cons:** More complex to implement; 3 round-trips during signup is UX friction.

---

### Approach 5 — Behavioral Timing Fingerprint (Signal, not Gate)

LLMs have inference latency that scales with output token count. Scripts respond in <10ms regardless of complexity.

- Require a "write 150 tokens explaining X" challenge
- Reject responses under 500ms (no LLM generates 150 tokens that fast)
- Flag instant responses for review

**Use as a secondary signal**, not a hard gate. Easy to add as middleware on the verify endpoint.

---

### Approach 6 — Progressive Trust Tiers (Most Pragmatic Path)

Don't gate registration entirely — instead, unlock capabilities based on verification level:

| Tier | Verification | Access |
|------|-------------|--------|
| Basic | Keypair + old nonce challenge | 10 API calls/day, read-only |
| **Agent** | Contextual challenge (Approach 2) | Full API access |
| Vouched | Human owner email code | Higher rate limits, org features |
| **Attested** | Model attestation (Approach 3) | Premium, SLA, no rate limits |

---

## Decision

**Implemented now:** Approach 2 (Contextual Self-Description)
**Roadmap:** Approach 3 (Model Attestation) — should be prioritized as a strategic differentiator
**Secondary signal:** Approach 5 (Timing) — add as passive monitoring
**Future:** Approach 6 (Tiers) once attestation is live

---

## Implementation Files

- `backend/src/routes/v1/agentAuth.ts` — registration + verification routes
- `backend/src/services/agentIdentityService.ts` — challenge generation + validation
- `backend/src/stores/agentSignupStore.ts` — stores `agentPurpose` + `challengeParams`
- `backend/src/stores/agentIdentityStore.ts` — stores `agentPurpose` in agent identity
- `backend/src/types/auth.ts` — `AgentSignup`, `AgentIdentity`, `ChallengeParams` types
