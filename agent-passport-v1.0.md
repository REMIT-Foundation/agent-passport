# Agent Passport
## Signed Capability Manifests for Autonomous AI Agents

**Cryptographic Identity, Attenuated Capabilities and Local Authority for REMIT-Compliant Agent Systems**

Version 1.0 | Published June 2026 | REMIT Foundation

Licensed under Creative Commons Attribution 4.0 International (CC BY 4.0).
Anyone may use, adapt, and build upon this specification freely, provided attribution is given.

---

## Foreword

REMIT tells operators what responsible agent behaviour looks like: a defined scope, real time evaluation, complete auditability and human oversight. But a machine-readable allowed profile on its own does not stop an agent from bypassing controls if the execution environment grants ambient authority.

Agent Passport addresses that gap. It is a signed capability manifest, bound to a cryptographic agent identity, issued by a passport authority and enforced at runtime through a mandatory mediation layer. Think of it as a TLS certificate for autonomous agents, except the certificate carries compile-time enforceable permissions rather than merely identifying a host.

Control does not live in the passport document. Control lives in the execution substrate the passport is bound to: the enforcement layer that intercepts every tool call, the sandbox that removes unreachable systems from the environment, and the broker that maps capabilities to specific endpoints and methods.

This specification defines the passport format, capability token model, authority operations (including fully local deployment), revocation, and how implementers bind passports to REMIT allowed profiles.

**The REMIT Foundation**
June 2026

---

## Contents

1. Introduction
2. Scope and Purpose
3. Definitions
4. Architecture Overview
5. Passport Manifest
6. Capability Tokens
7. Passport Authority
8. Local and Offline Authority
9. Enforcement Layer Requirements
10. Runtime Attestation
11. Revocation
12. Audit and REMIT Integration
13. Governance and Amendments

Appendix A: Example Passport Manifest
Appendix B: Example Capability Token
Appendix C: Local Authority Deployment Model
Appendix D: Mapping to REMIT and ATMS

---

## 1. Introduction

### 1.1 What Agent Passport Is

An Agent Passport is a verifiable bundle consisting of:

1. **Passport manifest** — a signed JSON document defining machine-enforceable policies: capability-to-tool mappings, data-plane restrictions, context constraints and explicit denials.
2. **Agent identity** — a cryptographic key pair (or certificate chain) identifying the agent instance or agent class.
3. **Capability tokens** — short-lived, attenuated, context-bound tokens authorising specific actions within the manifest.
4. **Revocation hooks** — mechanisms by which the authority or operator can invalidate a passport, identity or token before its natural expiry.

The passport is issued by a **passport authority**. The authority may be operated by the REMIT Foundation, a cloud provider, or — and this is a first-class deployment model — **entirely within the operator's own infrastructure**, including air-gapped environments.

### 1.2 What Agent Passport Is Not

Agent Passport is not:

- A replacement for REMIT runtime evaluation, audit logging or alerting.
- A network firewall. It is a **semantic execution firewall** that operates on tool calls, API invocations and data access requests.
- A guarantee of safety if the agent can bypass the enforcement layer or reach tools outside the brokered tool layer.
- A centralised identity provider requirement. Local authority is a normative deployment option, not an exception.

### 1.3 Relationship to REMIT

REMIT defines the **allowed profile** as the contract between operator and agent. Agent Passport is one conformant way to express, sign, distribute and bind that profile to a runtime.

| REMIT concept | Agent Passport equivalent |
|---|---|
| Allowed Profile (§5.2) | Passport manifest `capabilities` and `denies` |
| Session (§5.1) | Token `session_id` binding + manifest `session` block |
| Interception layer (§5.3.1) | Enforcement layer (§9) |
| Verdict Allow / Flag / Block | Enforcement decision + REMIT evaluation |
| Audit log (§5.4) | Signed action records referencing `passport_id` and `token_id` |
| ATMS M4 Cryptographic Agent Identity | Passport `identity` block |

Operators may implement REMIT without Agent Passport (rule-based profiles, unsigned JSON). Operators requiring high assurance, multi-agent delegation, or regulatory evidence of cryptographic binding should implement Agent Passport alongside REMIT.

### 1.4 Relationship to Other Standards

- **ATMS** — Passport mitigations map to M3 (Least Privilege Tool Access), M4 (Cryptographic Agent Identity) and M5 (Tamper Evident Audit Logging). See Appendix D.
- **REMIT Enforcement Profile** — Reference architectures for non-bypassable enforcement layers that consume passports and tokens.
- **SPIFFE / SPIRE, OAuth, macOS sandbox** — Agent Passport is compatible with existing identity and capability patterns; it does not mandate a specific PKI vendor.

---

## 2. Scope and Purpose

### 2.1 What This Specification Covers

- Structure and signing of passport manifests.
- Capability token format, issuance, attenuation and validation.
- Passport authority responsibilities, including local/offline operation.
- Requirements for enforcement layers that consume passports.
- Revocation models for connected and disconnected environments.
- Optional runtime attestation as a token issuance gate.
- Integration with REMIT sessions, allowed profiles and audit logs.

### 2.2 What This Specification Does Not Cover

- LLM prompt design, intent extraction algorithms or risk scoring models (covered by REMIT implementers).
- Specific vendor implementations of TEEs, service meshes or MCP servers (covered by REMIT Enforcement Profile).
- Cross-organisation federation protocols (deferred to a future Federation Profile).

---

## 3. Definitions

| Term | Meaning |
|---|---|
| Passport Authority | The trusted component that signs passport manifests, issues capability tokens and publishes revocation state. |
| Passport Manifest | The signed JSON policy document associated with an agent for a session or task. |
| Capability | A named, machine-enforceable permission such as `read_crm` that maps to concrete tool endpoints and methods. |
| Capability Token | A short-lived, signed authorisation for one or more capabilities within a specific context. |
| Enforcement Layer | The trusted runtime boundary that intercepts every agent action, validates tokens and blocks non-compliant execution. |
| Attenuation | Reducing the scope of a capability when delegating to a sub-agent or issuing a child token. A child token must never exceed the parent. |
| Ambient Authority | Access to resources without an explicit capability token (e.g. raw network, filesystem, shell). Agent Passport requires ambient authority to be eliminated or strictly bounded. |
| Local Authority | A passport authority operated entirely within the operator's infrastructure, with no dependency on external issuance services at runtime. |
| Revocation List | A signed, append-only list of revoked passport, identity or token identifiers. |

---

## 4. Architecture Overview

Agent systems implementing this specification must separate three trust zones.

### 4.1 Agent Runtime (Untrusted)

The LLM, reasoning layer and tool selection logic. This zone must be treated as capable of attempting any action. It must not hold long-lived credentials for external systems.

### 4.2 Enforcement Layer (Trusted Boundary)

The semantic execution firewall. It:

- Intercepts every tool call before execution.
- Validates the agent identity, passport manifest and capability token.
- Maps capabilities to allowed endpoints, methods and parameters.
- Blocks any action outside policy.
- Emits signed audit records to REMIT-compliant logging.

If the agent bypasses this layer, control is lost regardless of passport content.

### 4.3 Tool Layer (Brokered)

CRM APIs, databases, SSH gateways and internal services. Tools are exposed only through brokers registered with the enforcement layer. Systems that an agent must not reach must be **unreachable** from the agent runtime environment (network isolation, absent credentials, unregistered tools), not merely denied in policy text.

### 4.4 Authority (Off-Runtime)

Issues and signs manifests and tokens. Validates optional runtime attestations. Publishes revocation state. May run locally, in the cloud, or in a hybrid model (local issuance, optional federation).

```
┌─────────────────────────────────────────────────────────────┐
│  Passport Authority (local or hosted)                       │
│  - Sign manifest  - Issue tokens  - Publish revocation      │
└──────────────────────────┬──────────────────────────────────┘
                           │ issue at session start
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Agent Runtime (untrusted)                                  │
│  LLM → tool selection → [all calls go to enforcement layer] │
└──────────────────────────┬──────────────────────────────────┘
                           │ every tool call
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Enforcement Layer (trusted)                                │
│  validate identity + token → map capability → allow/deny    │
└──────────────────────────┬──────────────────────────────────┘
                           │ only on allow
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Tool Layer (brokered)                                      │
│  CRM API │ DB read broker │ (SSH not present / unroutable)  │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Passport Manifest

### 5.1 Manifest Requirements (Mandatory)

Every passport manifest must:

- Be JSON (UTF-8) conforming to the schema in Appendix A.
- Include a unique `passport_id` (ULID or UUID).
- Include an `identity` block referencing the agent's public key or certificate.
- Include a `session` block binding the passport to a REMIT session and task.
- Define one or more `capabilities` with compile-time enforceable rules.
- Define explicit `denies` that take precedence over capabilities.
- Be signed by the passport authority using a recognised algorithm (Ed25519 or ES256 minimum).
- Include `issued_at`, `expires_at` and `issuer` fields.
- Reference the REMIT `allowed_profile_id` it realises.

### 5.2 Capability Rules (Mandatory)

Each capability must be precise enough to compile into enforcement rules. Descriptive labels alone are non-compliant.

A compliant capability rule must specify at minimum:

| Field | Requirement |
|---|---|
| `name` | Stable identifier (e.g. `read_crm`) |
| `tools` | Registered tool identifiers in the enforcement layer |
| `resources` | URI patterns, API path patterns or broker identifiers |
| `methods` | Allowed HTTP methods, RPC operations or tool operations |
| `constraints` | Optional: row filters, field allowlists, rate limits, cost ceilings |

**Non-compliant:** `"read_crm": "Agent may read CRM data"`

**Compliant:** `"read_crm"` maps to `crm_api` tool, paths `/v1/customers/*`, methods `GET` only.

### 5.3 Deny Rules (Mandatory)

Deny rules must be explicit and take precedence over capabilities. Examples:

- Forbidden tools: `ssh_exec`, `code_execution`, `send_email`
- Forbidden network ranges: `10.0.0.0/8`, `192.168.0.0/16`
- Forbidden data classes: `credentials`, `pii_unmasked`
- Forbidden side effects: `write`, `delete`, `delegate_beyond_parent`

### 5.4 Context Constraints (Mandatory where applicable)

Manifests may include context constraints enforced at token validation time:

- `environment`: production, staging, development
- `network_zones`: allowed egress tags
- `caller_identity`: orchestrator or human principal who authorised the session
- `max_actions`, `max_cost`, `max_duration` session limits

---

## 6. Capability Tokens

### 6.1 Purpose

Capability tokens prevent scope inflation during execution. A token authorises a specific attenuated subset of the passport manifest for a bounded time and context.

Static scopes (e.g. "agent role: analyst") are insufficient. Tokens must be bound to:

- Specific capability name(s)
- Specific resource patterns (equal to or narrower than manifest)
- Specific session identifier
- Specific runtime identity (when attestation is used)
- Time window (`not_before`, `expires_at`)
- Optional: single-use (`jti` consumed on first successful enforcement)

### 6.2 Issuance (Mandatory)

- Only the passport authority (or a delegated sub-authority registered in the manifest) may issue tokens.
- Token signing keys may differ from manifest signing keys but must chain to the same trust root.
- Tokens must be issued after the passport manifest is validated and the REMIT session is created.
- Token lifetime must not exceed the passport `expires_at`.

### 6.3 Attenuation and Delegation (Mandatory)

When an agent delegates to a sub-agent:

1. The parent agent presents its capability token to the authority (or enforcement layer delegation endpoint).
2. The child receives a new token whose capabilities are a **strict subset** of the parent.
3. The child manifest or token must reference `parent_token_id` and `parent_agent_id`.
4. Delegation that would expand scope must be rejected.

This directly mitigates ATMS delegated authority abuse.

### 6.4 Validation (Mandatory)

The enforcement layer must validate on every tool call:

1. Token signature against authority trust root.
2. Token not expired and not before `not_before`.
3. Token not revoked (§11).
4. `session_id` matches active REMIT session.
5. Requested tool, resource and method are authorised by token.
6. No deny rule in the parent manifest is violated.
7. Context constraints (environment, network zone) match.

Validation must complete within the REMIT evaluation latency target (500ms at p95, including REMIT verdict computation).

---

## 7. Passport Authority

### 7.1 Responsibilities (Mandatory)

A passport authority must:

- Operate a root or intermediate CA for signing manifests and tokens.
- Provide a token issuance API (REST or gRPC) or offline issuance tooling.
- Publish its public keys / JWKS or certificate chain for enforcement layer verification.
- Maintain revocation state (§11).
- Log all issuance and revocation events to an append-only audit store independent of the agent runtime.
- Support binding manifests to REMIT `allowed_profile_id` and `session_id`.

### 7.2 Trust Root (Mandatory)

- Operators must pin the authority trust root in the enforcement layer configuration.
- Root key material must be protected (HSM, cloud KMS or hardware security module recommended).
- Key rotation must be supported without invalidating in-flight sessions (overlap publication window minimum 24 hours).

### 7.3 Deployment Models (Normative)

Three deployment models are defined. All are equally conformant.

| Model | Description | Typical use |
|---|---|---|
| Hosted | Authority operated by a third party or REMIT Foundation | SaaS agents, SMB |
| Local | Authority operated on operator infrastructure, may use outbound network only for optional updates | Enterprise, regulated |
| Air-gapped | Local authority with no runtime network dependency; revocation via local CRL distribution | Defense, critical infrastructure |

### 7.4 Public Registry (Recommended)

Operators that issue passports for external partners, or that wish to support federation, should list their passport authority in the REMIT Foundation **Passport Authority Registry**. Each listing includes:

- `issuer_uri` and optional `jwks_url`
- `trust_root_fingerprint` for offline trust pinning
- `deployment_model` (hosted, local, air_gapped)

The registry is published at `https://remit.foundation/agent-passport/registry` and as JSON at `/api/v1/agent-passport/authorities`. Listing is voluntary for purely internal local authorities but recommended when other parties must verify your signatures.

---

## 8. Local and Offline Authority

### 8.1 Design Intent

Most enterprises cannot depend on an external service to authorise agent actions. Data sovereignty, regulatory obligation and operational resilience require a **local passport authority** that operators run alongside their agent platform.

Local authority is not a degraded mode. It is the recommended deployment for high-assurance REMIT implementations.

### 8.2 Local Authority Requirements (Mandatory)

A local authority deployment must:

- Run entirely within the operator's security boundary.
- Not require connectivity to external issuance services at runtime.
- Store root keys in operator-controlled HSM or KMS.
- Issue manifests and tokens via internal API or signed batch files (for air-gapped flows).
- Distribute trust roots to enforcement layers via configuration management, not live fetch at request time.
- Produce audit logs suitable for REMIT §5.4 and regulatory retention requirements.

### 8.3 Air-Gapped Operation

In fully disconnected environments:

- Manifests and tokens may be pre-issued as signed files and loaded by the enforcement layer at session start.
- Revocation relies on **short token lifetimes** (recommended maximum 15 minutes) and locally distributed signed revocation lists.
- Operators must operate a local CRL sync process on removable media or one-way diodes where applicable.
- Real-time central revocation is not available; compensating controls are short TTL and session termination.

### 8.4 Optional Federation

Operators may additionally trust a central authority for cross-organisation agent collaboration. Federation is optional. Local authority remains the source of truth for internal agents.

Federation requirements are out of scope for v1.0.

---

## 9. Enforcement Layer Requirements

### 9.1 Mandatory Mediation

Every agent action — tool calls, API requests, data reads, code execution, network egress — must pass through the enforcement layer. No exceptions.

### 9.2 Non-Bypassability (Mandatory)

Implementers must demonstrate that the agent runtime cannot:

- Call tools directly without passing through the enforcement layer.
- Access network endpoints not mapped to an allowed capability.
- Read credentials or secrets not injected by the broker at execution time.
- Modify or delete its own audit log.

Recommended controls (see REMIT Enforcement Profile for reference architectures):

- SDK wrapper that refuses uninstrumented tool registration.
- Proxy or MCP server as sole tool endpoint.
- Container network policies matching manifest `denies`.
- seccomp / gVisor profiles removing shell and raw socket access.

### 9.3 Capability-to-Endpoint Mapping (Mandatory)

The enforcement layer must maintain a registry mapping:

```
capability name → tool broker → allowed resources + methods
```

Requests that match no mapping are denied by default (deny-by-default).

### 9.4 REMIT Verdict Integration (Mandatory)

The enforcement layer must invoke or embed REMIT evaluation (Allow / Flag / Block) and attach to each audit record:

- `passport_id`, `token_id`, `capability` used
- REMIT `verdict`, `score`, `reasoning`
- Cryptographic signature of the enforcement decision (recommended)

---

## 10. Runtime Attestation

### 10.1 Purpose

Runtime attestation binds capability tokens to a verified execution environment, preventing tokens issued for a hardened sandbox from being used in an unverified runtime.

### 10.2 Requirements (Optional — High Assurance Profile)

When attestation is enabled:

- The enforcement layer or sandbox must produce an attestation quote (TPM, TEE, or container image hash).
- The passport authority must verify the quote against a policy before issuing tokens.
- Tokens must include `runtime_attestation_hash` binding.
- The enforcement layer must reject tokens whose attestation binding does not match the current environment.

Attestation is optional for v1.0 baseline compliance. It is required for REMIT Level 4 (High Assurance) when that certification track is published.

---

## 11. Revocation

### 11.1 Revocable Objects

The authority must support revocation of:

- Entire passport (`passport_id`)
- Agent identity (`agent_id` / certificate serial)
- Individual capability token (`token_id` / `jti`)
- Authority intermediate keys (emergency)

### 11.2 Revocation Mechanisms

| Mechanism | Connected environments | Air-gapped environments |
|---|---|---|
| Signed CRL / revocation list | Required | Required (local distribution) |
| Online status API | Recommended | Not available |
| Short token TTL | Recommended | Required |
| Session termination via REMIT | Required | Required |

### 11.3 Enforcement Behaviour (Mandatory)

On revocation:

- New tokens must not be issued for revoked passports or identities.
- The enforcement layer must reject revoked tokens on the next validation (maximum propagation delay: 60 seconds in connected mode).
- Block verdict must trigger REMIT §5.5 alerting.
- Revocation events must be audit-logged with reason and operator principal.

---

## 12. Audit and REMIT Integration

### 12.1 Audit Record Extensions (Mandatory)

REMIT audit log entries (§5.4) must be extended with:

| Field | Description |
|---|---|
| `passport_id` | Manifest identifier |
| `token_id` | Capability token identifier |
| `capability` | Capability name exercised |
| `enforcement_decision` | allow or deny at passport layer |
| `authority_issuer` | Issuer URI of signing authority |

### 12.2 Dual-Layer Decisions

A complete decision record includes:

1. **Passport layer** — cryptographic allow/deny based on token and manifest.
2. **REMIT layer** — semantic Allow/Flag/Block based on allowed profile and risk.

Both layers may deny. If passport layer denies, the action must not execute regardless of REMIT verdict.

### 12.3 Anomaly and Feedback Loop (Recommended)

Operators should monitor for:

- Repeated passport denials (possible bypass attempts).
- Token reuse from unexpected runtime identities.
- Capability usage patterns diverging from policy graph.
- Revocation triggers linked to ATMS threat categories.

---

## 13. Governance and Amendments

Agent Passport is maintained by the REMIT Foundation under the same governance process as the REMIT Core Specification. Semantic versioning applies. Implementers must document which version they implement.

### 13.1 Contact

- Specification questions: spec@remitframework.org
- Certification enquiries: certification@remitframework.org

---

## Appendix A: Example Passport Manifest

```json
{
  "passport_version": "1.0",
  "passport_id": "ppt_01k0abc123def456",
  "allowed_profile_id": "profile_01j9za9k0l1m2n3o4p5q6r7s",
  "issuer": "https://passport.authority.corp.example",
  "issued_at": "2026-06-23T09:00:00Z",
  "expires_at": "2026-06-23T17:00:00Z",
  "identity": {
    "agent_id": "agent_sales_summariser_v2",
    "public_key": "base64url-ed25519-public-key",
    "key_algorithm": "Ed25519"
  },
  "session": {
    "session_id": "sess_01j9zk2a3b4c5d6e7f8g9h0i",
    "org_id": "org_01j9za1b2c3d4e5f6g7h8i9j",
    "task": "Summarise total revenue by product for Q1 2026."
  },
  "capabilities": [
    {
      "name": "read_crm",
      "tools": ["crm_api"],
      "resources": ["crm_api:/v1/customers/*", "crm_api:/v1/orders/*"],
      "methods": ["GET"],
      "constraints": {
        "fields_allowlist": ["id", "name", "revenue", "product_id", "order_date"]
      }
    },
    {
      "name": "text_output",
      "tools": ["text_generation"],
      "resources": ["text_generation:/complete"],
      "methods": ["POST"],
      "constraints": {
        "max_output_tokens": 2000
      }
    }
  ],
  "denies": [
    { "type": "tool", "value": "ssh_exec" },
    { "type": "tool", "value": "send_email" },
    { "type": "tool", "value": "code_execution" },
    { "type": "network", "value": "10.0.0.0/8" },
    { "type": "network", "value": "192.168.0.0/16" },
    { "type": "data_class", "value": "credentials" }
  ],
  "context_constraints": {
    "environment": "production",
    "max_actions": 50,
    "max_duration_seconds": 3600
  },
  "signature": {
    "algorithm": "Ed25519",
    "key_id": "corp-passport-root-2026",
    "value": "base64url-signature"
  }
}
```

---

## Appendix B: Example Capability Token

JWT-style claims (signed compact serialization or JSON with detached signature):

```json
{
  "iss": "https://passport.authority.corp.example",
  "sub": "agent_sales_summariser_v2",
  "jti": "tok_01k0def789ghi012",
  "passport_id": "ppt_01k0abc123def456",
  "session_id": "sess_01j9zk2a3b4c5d6e7f8g9h0i",
  "capabilities": ["read_crm"],
  "resources": ["crm_api:/v1/customers/*"],
  "methods": ["GET"],
  "nbf": 1719133200,
  "exp": 1719134100,
  "runtime_attestation_hash": null
}
```

---

## Appendix C: Local Authority Deployment Model

### C.1 Recommended Components

| Component | Role |
|---|---|
| Passport CA | Root and intermediate keys (HSM-backed) |
| Issuance API | Session-start manifest and token issuance |
| CRL publisher | Signed revocation lists to enforcement layers |
| Audit store | Append-only issuance and revocation log |
| Admin console | Profile binding, emergency revoke, key rotation |

### C.2 Bootstrap Sequence

1. Operator generates root CA and distributes trust anchor to enforcement layers.
2. REMIT session created; allowed profile derived from task.
3. Local authority signs passport manifest bound to `session_id` and `allowed_profile_id`.
4. Authority issues short-lived capability tokens.
5. Enforcement layer validates on every tool call until session termination or revocation.

### C.3 Comparison: Local vs Hosted Authority

| Concern | Local authority | Hosted authority |
|---|---|---|
| Data sovereignty | Full | Depends on provider |
| Air-gap support | Yes | No |
| Operational burden | PKI ops on operator | Lower |
| Cross-org federation | Requires explicit trust setup | Easier |
| Revocation speed | CRL sync dependent | Real-time possible |

---

## Appendix D: Mapping to REMIT and ATMS

### REMIT Principle Mapping

| REMIT Principle | Agent Passport contribution |
|---|---|
| P1 Defined Scope | Signed manifest compiles to enforceable rules |
| P2 Real Time Enforcement | Token validation on every intercepted action |
| P3 Least Privilege | Attenuated tokens; deny-by-default |
| P4 Complete Auditability | Signed decisions; passport fields in audit log |
| P5 Human Oversight | Revocation and session termination hooks |
| P6 Transparency | `identity` and `task` in manifest |
| P7 Tenant Isolation | Per-org trust roots and local authority |

### ATMS Mitigation Mapping

| ATMS Mitigation | Agent Passport section |
|---|---|
| M3 Least Privilege Tool Access | §5.2, §6.3 attenuation |
| M4 Cryptographic Agent Identity | §5.1 identity block, §7 |
| M5 Tamper Evident Audit Logging | §12 |
| Delegated authority abuse | §6.3 |
| Autonomy escalation | §5.3 denies, §6 tokens |

### OWASP Agentic AI Top 10

| Risk | Coverage |
|---|---|
| ASI02 Tool Misuse | Strong — endpoint-level capability mapping |
| ASI03 Identity and Privilege Abuse | Strong — cryptographic identity + attenuation |
| ASI10 Rogue Agents | Strong — mandatory mediation + revocation |
| ASI07 Inter-Agent Communication | Partial — delegation rules only; messaging out of scope |
