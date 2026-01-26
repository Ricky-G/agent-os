# RFC-001: IATP - Inter-Agent Trust Protocol

> A Cryptographic Trust Protocol for Multi-Agent Systems

**Status:** Draft  
**Authors:** Imran Siddique (Microsoft)  
**Created:** 2026-01-26  
**Target:** AAIF Interoperability Standards

---

## Abstract

This RFC specifies the Inter-Agent Trust Protocol (IATP), a cryptographic framework enabling secure, verifiable communication between autonomous AI agents. IATP provides "TLS for AI Agents"—establishing identity, trust levels, and policy enforcement at the protocol layer rather than in application code.

As multi-agent systems scale from single orchestrators to swarms of 100+ autonomous agents, the need for standardized trust mechanisms becomes critical. IATP addresses this by providing:

1. **Agent Attestation**: Cryptographic proof that agents run verified code
2. **Capability Discovery**: Standardized manifests advertising agent capabilities
3. **Trust Scoring**: Network-wide reputation tracking with automatic slashing
4. **Policy Enforcement**: Deterministic blocking of dangerous operations

## 1. Introduction

### 1.1 Problem Statement

Current multi-agent architectures suffer from a "trust void":

| Problem | Impact |
|---------|--------|
| No standard identity | Agents can't verify who they're talking to |
| No capability discovery | Agents don't know what others can do |
| No reputation system | Misbehaving agents continue to be trusted |
| Trust logic in agent code | Tight coupling, fragile systems |
| No audit trail | Accountability impossible |

### 1.2 Solution Overview

IATP extracts trust concerns into a **sidecar proxy** (inspired by Envoy/Istio for microservices):

```
┌─────────────────────────────────────────────────────────────────┐
│                        TRADITIONAL APPROACH                      │
│                                                                  │
│   Agent A ──── untrusted network ────> Agent B                  │
│     │                                     │                      │
│   (trust logic embedded)          (trust logic embedded)        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                          IATP APPROACH                           │
│                                                                  │
│   Agent A ──> IATP Sidecar A ═══════> IATP Sidecar B ──> Agent B│
│                    │                       │                     │
│              (handles trust)         (handles trust)            │
│                                                                  │
│   Agents are simple functions. Infrastructure handles security. │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Design Goals

1. **Zero Configuration**: Works out-of-box with sensible defaults
2. **Progressive Trust**: Support trust levels from `untrusted` to `verified_partner`
3. **Fail Safe**: Block dangerous operations (e.g., PII to untrusted agents)
4. **Auditable**: Every decision logged with trace IDs
5. **Interoperable**: Works with MCP, AGENTS.md, and existing frameworks

## 2. Protocol Specification

### 2.1 The Three-Phase Handshake

```
┌─────────┐                                    ┌─────────┐
│ Agent A │                                    │ Agent B │
│ Sidecar │                                    │ Sidecar │
└────┬────┘                                    └────┬────┘
     │                                              │
     │ Phase 1: Capability Discovery                │
     │ GET /.well-known/agent-manifest              │
     ├─────────────────────────────────────────────>│
     │                                              │
     │ 200 OK + CapabilityManifest                  │
     │<─────────────────────────────────────────────┤
     │                                              │
     │ Phase 2: Attestation Verification            │
     │ GET /.well-known/agent-attestation           │
     ├─────────────────────────────────────────────>│
     │                                              │
     │ 200 OK + Attestation (signed by Control Plane)│
     │<─────────────────────────────────────────────┤
     │                                              │
     │ Phase 3: Policy Evaluation                   │
     │ (Local: Does manifest meet our policies?)    │
     │                                              │
     │ Phase 4: Execution or Block                  │
     │ POST /proxy {task, data, trace_id}           │
     ├─────────────────────────────────────────────>│
     │                                              │
     │ 200 OK / 449 Warning / 403 Blocked           │
     │<─────────────────────────────────────────────┤
```

### 2.2 Capability Manifest

Every IATP-compliant agent MUST expose:

```
GET /.well-known/agent-manifest
```

**Required Fields:**

```json
{
  "$schema": "https://agent-os.dev/iatp/v1/manifest.schema.json",
  "identity": {
    "agent_id": "carbon-auditor-v1.2.0",
    "verification_key": "ed25519:abc123...",
    "owner": "Climate Corp",
    "contact": "security@climatecorp.com"
  },
  "trust_level": "verified_partner",
  "capabilities": {
    "idempotency": true,
    "max_concurrency": 10,
    "sla_latency_ms": 2000
  },
  "reversibility": {
    "level": "full",
    "undo_window_seconds": 3600
  },
  "privacy": {
    "retention_policy": "ephemeral",
    "human_in_loop": false,
    "training_consent": false
  },
  "protocol_version": "1.0"
}
```

### 2.3 Agent Attestation

Cryptographic proof that an agent runs verified code:

```
GET /.well-known/agent-attestation
```

**Response:**

```json
{
  "agent_id": "carbon-auditor-v1.2.0",
  "codebase_hash": "sha256:a3f5b9c2d1e0f...",
  "config_hash": "sha256:d8e2a1b7c6f5...",
  "signature": "base64_encoded_signature",
  "signing_key_id": "control-plane-prod-2026",
  "timestamp": "2026-01-26T10:00:00Z",
  "expires_at": "2026-01-27T10:00:00Z"
}
```

**Verification Flow:**

1. Fetch attestation from remote agent
2. Verify signature using Control Plane's public key
3. Check `codebase_hash` matches expected value
4. Verify attestation hasn't expired
5. If valid, proceed; if invalid, reject with SIGKILL

### 2.4 Trust Scoring

**Base Scores:**

| Trust Level | Base Score |
|-------------|------------|
| `verified_partner` | 10 |
| `trusted` | 7 |
| `standard` | 5 |
| `unknown` | 2 |
| `untrusted` | 0 |

**Modifiers:**

| Condition | Modifier |
|-----------|----------|
| `reversibility != "none"` | +2 |
| `retention_policy == "ephemeral"` | +1 |
| `retention_policy == "permanent"` | -1 |
| `human_in_loop == true` | -2 |
| `training_consent == true` | -1 |

**Policy Decisions:**

| Trust Score | Action |
|-------------|--------|
| ≥ 7 | ✅ Allow |
| 3-6 | ⚠️ Warning (449 status, requires override) |
| < 3 | ⚠️ Warning (449 status, requires override) |
| PII + non-ephemeral | 🚫 Block (403 Forbidden) |

### 2.5 Reputation Slashing

When CMVK detects hallucination or misbehavior:

```http
POST /reputation/{agent_id}/slash
Content-Type: application/json

{
  "reason": "hallucination",
  "severity": "high",
  "trace_id": "abc123",
  "evidence": {
    "cmvk_drift_score": 0.45,
    "threshold": 0.15
  }
}
```

**Severity Penalties:**

| Severity | Score Reduction |
|----------|-----------------|
| `critical` | -2.0 |
| `high` | -1.0 |
| `medium` | -0.5 |
| `low` | -0.25 |

Reputation propagates across the network with conservative merge (minimum score wins).

## 3. MCP Integration

### 3.1 IATP as MCP Resource

IATP exposes agent certificates and trust information as MCP resources:

```
Resource URI: iatp://agent-id/manifest
Resource URI: iatp://agent-id/attestation
Resource URI: iatp://agent-id/reputation
```

**MCP Resource Template:**

```json
{
  "uri": "iatp://{agent_id}/manifest",
  "name": "Agent Capability Manifest",
  "mimeType": "application/json",
  "description": "IATP capability manifest for the specified agent"
}
```

### 3.2 MCP Tool for Trust Verification

```json
{
  "name": "iatp_verify",
  "description": "Verify trust relationship with another agent",
  "inputSchema": {
    "type": "object",
    "properties": {
      "remote_agent_id": {
        "type": "string",
        "description": "Agent ID to verify"
      },
      "required_trust_level": {
        "type": "string",
        "enum": ["verified_partner", "trusted", "standard"],
        "description": "Minimum required trust level"
      },
      "required_scopes": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Required capability scopes"
      }
    },
    "required": ["remote_agent_id"]
  }
}
```

### 3.3 Governed MCP Runtime

IATP wraps MCP tool calls with trust verification:

```python
# Before: Raw MCP call (no trust)
result = await mcp_client.call_tool("book_flight", {"dest": "NYC"})

# After: IATP-governed MCP call
result = await iatp_runtime.call_tool(
    tool="book_flight",
    args={"dest": "NYC"},
    remote_agent="booking-agent",
    min_trust_level="trusted"  # IATP enforces this
)
```

## 4. Security Considerations

### 4.1 Threat Model

| Threat | Mitigation |
|--------|------------|
| Agent impersonation | Attestation with signed codebase hash |
| Replay attacks | Nonce + timestamp in attestation |
| Trust score gaming | Conservative network merge |
| Data exfiltration | PII detection + ephemeral retention |
| Man-in-the-middle | TLS + attestation signatures |

### 4.2 Sensitive Data Detection

Sidecars MUST detect and protect:

- Credit card numbers (Luhn validation)
- SSNs (pattern matching)
- API keys/tokens (entropy detection)

### 4.3 Privacy Scrubbing

All logged data MUST be scrubbed:
- Credit cards → `[CREDIT_CARD_REDACTED]`
- SSNs → `[SSN_REDACTED]`
- Tokens → `[TOKEN_REDACTED]`

## 5. Implementation

### 5.1 Reference Implementation

The reference implementation is available at:
- Package: `inter-agent-trust-protocol` (PyPI)
- Source: `github.com/imran-siddique/agent-os/packages/iatp`

### 5.2 Minimal Compliance

An IATP-compliant implementation MUST:

1. Expose `/.well-known/agent-manifest`
2. Validate incoming requests against manifest
3. Generate unique trace IDs (UUID v4)
4. Log all transactions with privacy scrubbing

### 5.3 Full Compliance

Full compliance adds:

- Agent attestation verification
- Reputation tracking and slashing
- MCP resource integration
- OpenTelemetry tracing

## 6. AAIF Alignment

### 6.1 Complementary to MCP

| Layer | Standard | Purpose |
|-------|----------|---------|
| Connectivity | MCP | Tools, prompts, resources |
| Instructions | AGENTS.md | Agent behavior |
| **Trust** | **IATP** | **Inter-agent security** |
| Safety | Agent OS Kernel | Policy enforcement |

### 6.2 Proposed AAIF Integration

IATP could be contributed to AAIF as the **agent-to-agent trust standard**, complementing MCP's tool connectivity and AGENTS.md's behavioral specification.

**Value to AAIF:**
- Fills the "zero trust" gap in multi-agent systems
- Enables secure agent federation across organizations
- Provides auditable trust decisions for compliance

## 7. References

- [IATP Specification](../../packages/iatp/spec/001-handshake.md)
- [MCP Protocol](https://modelcontextprotocol.io/)
- [AGENTS.md Specification](https://github.com/OpenAI/agents-spec)
- [Envoy Proxy](https://www.envoyproxy.io/)
- [Agent OS Kernel](../../docs/kernel-internals.md)

---

## Appendix A: Example Scenarios

### A.1 Energy Grid Trading

100 distributed energy resource (DER) agents negotiate power trades:

```
Solar Agent A ──IATP──> Grid Operator ──IATP──> Battery Agent B
     │                       │                       │
   (verified_partner)    (trust=10)           (verified_partner)
```

All agents verify attestations before trading. If any agent's code hash doesn't match, trade is blocked.

### A.2 DeFi Risk Detection

Trading agent signals risk sentinel:

```
Trading Agent ──IATP──> Risk Sentinel
      │                      │
  (trust=8)              (issues SIGKILL)
```

Sentinel has permission to halt trading agent via IATP signal mechanism.

### A.3 Healthcare Consultation

Doctor agent consults specialist:

```
Doctor Agent ──IATP──> Specialist Agent
      │                      │
  (PII: patient data)   (ephemeral retention required)
```

IATP blocks request if specialist has permanent data retention.

---

**Document Status:** Draft - Ready for AAIF review
