# Production Status

> **ExecutionProof Pre-Execution Governance Layer — production status**

This document exists so a reviewer never has to guess what is real. It states, explicitly, what is
**implemented**, what is **not yet built**, and what is **pilot-configuration scope**. We treat honest
disclosure as a trust feature, not a liability. Our claims are kept narrower than our evidence.

**Scope of "implemented" in this table:** the capability exists in RF's production service codebase.
**Public, independently reachable availability of the hosted `/v2` endpoints is NOT established from this public
repository** — at last check the public endpoints returned `404` (see
[Public endpoint availability](#public-endpoint-availability) below). Do not read "implemented" as "the public
hosted API is live" or as independent verification of the live AWS KMS-backed signing path.

_Status as of the v1.0.0-public release (August 2026); public-endpoint check dated inline below._

---

## Summary

| Capability | Status |
|---|---|
| ALLOW / HOLD / DENY decision engine (four-engine pipeline) | ✅ **Implemented** |
| ProofRecord™ generation with ML-DSA-65 signing (AWS KMS-backed design) | ✅ **Implemented** · ⚠️ live AWS KMS-backed path not independently verified here |
| HMAC integrity layer + hash-chained ProofRecords | ✅ **Implemented** |
| M-of-N approval **state machine** (API/state level) | ✅ **Implemented** |
| Replay protection (nonce enforcement, `EP-7008`) | ✅ **Implemented** |
| Payment authorization boundary enforcement (first production boundary) | ✅ **Implemented** |
| Independent ProofRecord verification (public PQ key endpoint) | ⚠️ **Public endpoint currently returns 404** (see note) |
| Notification / alert delivery (email, SMS, chat, pager) | ❌ **Not built** |
| Identity-provider / SSO integration, approval inbox | ❌ **Not built** |
| Approval-routing UI (review & approve HELD actions) | ❌ **Not built** |
| Formal RF-100 §8.4 conformance certification | ⏳ **Pending independent external review** |
| Eight-engine Enterprise pipeline (Coherence Envelope, Gate Layers, Rail Classifier, Rail Enforcement) | 🔧 **Enterprise / pilot-config scope** |
| Additional boundaries (infrastructure, data release, access, deploy) beyond payments | 🔧 **Pilot-config scope** |

Legend: ✅ implemented in RF's codebase · ⚠️ availability/independent verification not established from this public repo · ❌ not yet built · ⏳ pending external review · 🔧 available under pilot / enterprise configuration.

---

## What IS implemented

These behaviors are implemented in RF's production service codebase. "Implemented" here means the capability
exists in that codebase; it is **not** a claim that the public hosted API is independently reachable (see
[Public endpoint availability](#public-endpoint-availability)) or that the live AWS KMS-backed signing path has
been independently verified from this repository.

### Decision engine — ALLOW / HOLD / DENY
The four-engine sequential pipeline (**Authority → Evidence → Constraint → Control**) runs on every request,
short-circuits on the first failure, and returns exactly one of **ALLOW / HOLD / DENY**. See
[ARCHITECTURE.md §2](ARCHITECTURE.md#2-the-four-engine-pipeline).

### ProofRecord generation and signing
Every decision — including DENY and HOLD — emits a hash-chained **ProofRecord™** signed with **ML-DSA-65
(FIPS 204) post-quantum signatures** plus an HMAC integrity layer. RF's production signing is **designed around
AWS KMS-backed keys**; the live AWS KMS-backed signing path is **not independently verified from this
repository**. ProofRecords are designed to be verified independently against the published ML-DSA-65 public key
(`GET /v2/crypto/pq-public-key`); note that at last check this **public key endpoint returned `404`** (see
[Public endpoint availability](#public-endpoint-availability)).

### M-of-N approval state machine
A HOLD parks an action in an approval state machine requiring **M** approvals from **N** eligible approvers.
Approvals are **submitted to** the API (`POST /v2/boundaries/{boundaryId}/approve`); distinct-approver and
eligibility rules are enforced (`EP-7006`). This is **state-machine and API level only** — see "not built"
below for what surrounds it.

### Replay protection
Single-use nonces are enforced by the Evidence engine; reuse or expiry returns **`EP-7008`**. Control-engine
idempotency keys prevent double-execution of the same intent.

### Payment authorization boundary
`payment-auth-prod` is the **first production boundary target** — the lighthouse boundary. It enforces
authority, evidence, amount/rate/allow-list constraints, and control-state safety for payment authorization
requests, then hands ALLOW results to a **downstream execution/rail partner** (e.g. Stripe) that actually
moves the money. ExecutionProof does not move money and does not replace payment rails.

## What is NOT yet built

Stated plainly. These are real gaps, not roadmap marketing.

### Notification / alert delivery
ExecutionProof does **not** send email, SMS, chat, or pager notifications, and does **not** "automatically
escalate to" human approvers. A HOLD is observable via the API; **routing and notification are the caller's
responsibility.**

### Identity integration (IdP / SSO / approval inbox)
There is **no** built-in identity-provider or SSO integration and **no** approval inbox. Callers authenticate
to the API and manage their own principals and approver identities.

### Approval-routing UI
There is **no** shipped user interface for reviewing and approving HELD actions. The approval capability is
API/state-level only.

### Formal RF-100 §8.4 conformance
The ProofRecord signing scheme follows the RF-100 profile, but **formal RF-100 §8.4 conformance remains
pending independent external review.** We do not claim certified conformance.

## Pilot / Enterprise configuration scope

Available under a configured pilot or the Enterprise tier — not part of the generally-available four-engine
core documented in this repository.

- **Eight-engine Enterprise pipeline** — adds Coherence Envelope, Gate Layers, Rail Classifier, and Rail
  Enforcement on top of the four core engines.
- **Additional deployment boundaries** — infrastructure commands, data release, access changes, and deployment
  operations use the same architecture as payments and are enabled per pilot. The
  [infrastructure-command example](examples/infrastructure-command/) shows the shape.
- **Pilot API access** — Pilot integrations are scoped directly with Remnant Fieldworks. Deployment route,
  authentication, and environment are confirmed per pilot. Public hosted-API availability is **not established
  from this repository** — at last check the public `/v2` endpoints returned `404` (see
  [Public endpoint availability](#public-endpoint-availability)). See
  [executionproof.io](https://executionproof.io).

## Public endpoint availability

This section records what can and cannot be confirmed about the **public** hosted API from outside RF, so that
"implemented" is never mistaken for "publicly live."

As of a **read-only check dated 2026-10-16**, the public `/v2` endpoints were **not reachable**:

| Endpoint (public) | Observed HTTP status |
|---|---|
| `GET https://executionproof.io/v2/version` | `404` |
| `GET https://executionproof.io/v2/crypto-manifest` | `404` |
| `GET https://executionproof.io/v2/crypto/pq-public-key` | `404` |
| `GET https://executionproof.io/` (site root) | `200` (marketing site; no `/v2` API mounted) |

What this means:

- The decision engine, ProofRecord signing, and other capabilities marked **Implemented** above describe RF's
  production service **codebase**, not a publicly reachable endpoint.
- Independent, offline ProofRecord verification depends on the public ML-DSA-65 key endpoint, which currently
  returns `404`. Until that endpoint is publicly served (or a key is otherwise published), public independent
  verification is **not currently possible from this repository alone**.
- This availability gap is tracked in the companion testbeds repository (see
  [executionproof-testbeds issue #25](https://github.com/derekhone/executionproof-testbeds/issues/25)).

Pilot integrations are scoped directly with Remnant Fieldworks; deployment route, authentication, and
environment are confirmed per pilot and are not represented by the public endpoint status above.

## Evidence and reproducibility

The claims above are backed by published, preregistered evidence:

- **253 case records: 250 PASS, 2 FAIL, 1 GATE-STOP.**
- **75 preregistered experiments.**
- Datasets archived in the [Remnant Fieldworks Zenodo community](https://zenodo.org/communities/remnant-fieldworks).
- Governing standard: [RF-100 v1.0](https://github.com/derekhone/rf-100) (public review).

The methods described are the subject of **8 non-provisional patent applications pending**. No patents have
been granted.

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
