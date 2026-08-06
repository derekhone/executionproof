# Production Status

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

This document exists so a reviewer never has to guess what is real. It states, explicitly, what is **live in
production**, what is **not yet built**, and what is **pilot-configuration scope**. We treat honest disclosure
as a trust feature, not a liability. Our claims are kept narrower than our evidence.

_Status as of the v1.0.0-public release (August 2026)._

---

## Summary

| Capability | Status |
|---|---|
| ALLOW / HOLD / DENY decision engine (four-engine pipeline) | ✅ **Live** |
| ProofRecord™ generation with hardware-backed ML-DSA-65 signing (AWS KMS) | ✅ **Live** |
| HMAC integrity layer + hash-chained ProofRecords | ✅ **Live** |
| M-of-N approval **state machine** (API/state level) | ✅ **Live** |
| Replay protection (nonce enforcement, `EP-7008`) | ✅ **Live** |
| Payment authorization boundary enforcement (first production boundary) | ✅ **Live** |
| Independent ProofRecord verification (public PQ key endpoint) | ✅ **Live** |
| Notification / alert delivery (email, SMS, chat, pager) | ❌ **Not built** |
| Identity-provider / SSO integration, approval inbox | ❌ **Not built** |
| Approval-routing UI (review & approve HELD actions) | ❌ **Not built** |
| Formal RF-100 §8.4 conformance certification | ⏳ **Pending independent external review** |
| Eight-engine Enterprise pipeline (Coherence Envelope, Gate Layers, Rail Classifier, Rail Enforcement) | 🔧 **Enterprise / pilot-config scope** |
| Additional boundaries (infrastructure, data release, access, deploy) beyond payments | 🔧 **Pilot-config scope** |

Legend: ✅ live · ❌ not yet built · ⏳ pending external review · 🔧 available under pilot / enterprise configuration.

---

## What IS live

These behaviors are implemented and running in the production service.

### Decision engine — ALLOW / HOLD / DENY
The four-engine sequential pipeline (**Authority → Evidence → Constraint → Control**) runs on every request,
short-circuits on the first failure, and returns exactly one of **ALLOW / HOLD / DENY**. See
[ARCHITECTURE.md §2](ARCHITECTURE.md#2-the-four-engine-pipeline).

### ProofRecord generation and signing
Every decision — including DENY and HOLD — emits a hash-chained **ProofRecord™** signed with **hardware-backed
ML-DSA-65 post-quantum signing via AWS KMS** plus an HMAC integrity layer. The ML-DSA-65 public key is
retrievable via `GET /v2/crypto/pq-public-key` for offline, independent verification.

### M-of-N approval state machine
A HOLD parks an action in an approval state machine requiring **M** approvals from **N** eligible approvers.
Approvals are **submitted to** the API (`POST /v2/boundaries/{boundaryId}/approve`); distinct-approver and
eligibility rules are enforced (`EP-7006`). This is **state-machine and API level only** — see "not built"
below for what surrounds it.

### Replay protection
Single-use nonces are enforced by the Evidence engine; reuse or expiry returns **`EP-7008`**. Control-engine
idempotency keys prevent double-execution of the same intent.

### Payment authorization boundary
`payment-auth-prod` is the **first production deployment boundary** — the lighthouse deployment. It enforces
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
- **Live API access** — the hosted `/v2` endpoints are available under **pilot access**; see
  [executionproof.io](https://executionproof.io).

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
