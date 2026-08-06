# Architecture

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

This document describes the core architecture of ExecutionProof: the four-engine decision pipeline, the
ProofRecord™ format and signing, ALLOW / HOLD / DENY semantics, the M-of-N HOLD approval flow, the deployment
boundary model, the threat model, and the honest production limitations of the current release.

For a high-level introduction, start with the [README](README.md). For rendered diagrams, see
[docs/architecture-diagram.md](docs/architecture-diagram.md). For the wire contract, see
[docs/openapi.yaml](docs/openapi.yaml).

---

## 1. Position in the stack

ExecutionProof is an **independent pre-execution governance layer**. It is called *after* an AI agent has
decided what it wants to do, and *before* that intent reaches the downstream system that would execute it.

```
AI Agent / Orchestrator  ──action request──►  ExecutionProof  ──verdict + ProofRecord──►  Downstream System
                                                                                          (payment rail, infra
                                                                                           control plane, data
                                                                                           store, IAM, etc.)
```

ExecutionProof does **not** execute the action itself. It decides *whether the action should be allowed to
execute* and produces evidence of that decision. Execution remains the responsibility of the downstream
system (for payments, that is a rail such as Stripe — a downstream execution partner, not a competitor).

## 2. The four-engine pipeline

Every request flows through four engines **in strict order**. The pipeline **short-circuits**: the first
engine that fails halts evaluation and determines the verdict. This ordering is deliberate — cheaper,
higher-authority checks run first, and no later engine can override an earlier failure.

| # | Engine | Question it answers | Failure code |
|---|---|---|---|
| 1 | **Authority** | Is the requesting principal authorized to perform *this class* of action on *this boundary*? | `EP-7001` |
| 2 | **Evidence** | Is the action backed by the required, valid, non-replayed evidence (context, approvals, nonce)? | `EP-7002` (invalid/missing), `EP-7008` (replay) |
| 3 | **Constraint** | Does the action stay within configured limits (amount, rate, allow-list, blast radius)? | `EP-7003` |
| 4 | **Control** | Is the execution state safe (idempotency, quorum satisfied, no conflicting in-flight action)? | `EP-7004` |

### Short-circuit semantics

- Engines execute Authority → Evidence → Constraint → Control.
- On the first failure, evaluation stops. No subsequent engine runs.
- The verdict is **DENY** for a hard failure, or **HOLD** when the failing condition is a policy that requires
  an M-of-N approval quorum rather than an outright rejection (typically surfaced by the Constraint or Control
  engine — e.g. "amount exceeds auto-approve threshold").
- A ProofRecord is emitted for **every** outcome, including DENY and HOLD. A denied action is still a verified
  decision with evidence.

### Enterprise configuration

The generally-available core documented here is four engines. The **Enterprise** configuration extends the
pipeline to eight engines, adding a **Coherence Envelope**, **Gate Layers**, a **Rail Classifier**, and **Rail
Enforcement**. Those engines are out of scope for this repository.

## 3. ALLOW / HOLD / DENY

| Verdict | Definition | Downstream expectation |
|---|---|---|
| **ALLOW** | All four engines passed. | The caller may execute. The ProofRecord proves the action was verified. |
| **HOLD** | Not rejected, but blocked pending an **M-of-N approval quorum**. | The caller must not execute until the HOLD clears. The ProofRecord records the pending state and quorum policy. |
| **DENY** | The action must not execute. | The caller must not execute. The ProofRecord records the failing engine and code. |

These three tokens are the **only** verdicts. There is no "maybe", no numeric score exposed as a verdict, and
no silent pass-through.

## 4. The M-of-N HOLD approval flow

A **HOLD** means the action is parked in an **approval state machine** that requires **M** approvals out of
**N** eligible approvers before it may proceed.

Important boundaries of responsibility:

- ExecutionProof **exposes the state machine**. Approvals are **submitted to it** via
  `POST /v2/boundaries/{boundaryId}/approve`. It tracks which distinct approvers have approved, enforces that
  the quorum is met, and enforces that approvers are distinct and eligible.
- ExecutionProof does **not** notify, page, email, or "automatically escalate to" human approvers. It does not
  discover who should approve. **Routing and notification are out of scope** for this layer and are **not yet
  built** (see [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md)). Callers integrate their own routing/inbox and
  submit approvals to the API.
- When the M-of-N quorum is satisfied, the HOLD clears and the action becomes eligible for execution. A new
  ProofRecord captures the cleared state, including the set of approver identities and their approval
  references.

```
verify -> HOLD (quorum 2-of-3 required)
      -> approve (approver A)   -> still HELD (1/2)
      -> approve (approver B)   -> quorum met -> eligible for ALLOW/execute
      -> approve (approver A again) -> rejected: distinct-approver rule (EP-7006)
```

Attempts to satisfy a quorum with a duplicate approver, an ineligible approver, or an approval for a boundary
the approver is not scoped to are rejected with `EP-7006`.

## 5. ProofRecord™

A **ProofRecord** is the tamper-evident receipt of a single verification decision. It is designed so that a
party who does **not** trust the caller can verify, after the fact, that a specific decision was made about a
specific action at a specific time.

### 5.1 Structure (informative)

```jsonc
{
  "proofId": "ep_pr_01J...",              // stable, unique identifier
  "boundaryId": "payment-auth-prod",       // deployment boundary
  "verdict": "ALLOW",                      // ALLOW | HOLD | DENY
  "decision": {
    "shortCircuitEngine": null,            // engine that halted the pipeline, or null on ALLOW
    "engineResults": [                     // per-engine pass/fail
      { "engine": "authority",  "result": "PASS" },
      { "engine": "evidence",   "result": "PASS" },
      { "engine": "constraint", "result": "PASS" },
      { "engine": "control",    "result": "PASS" }
    ],
    "errorCode": null                      // EP-7001..EP-7008 on non-ALLOW, else null
  },
  "request": {
    "actionDigest": "sha256:...",          // digest of the canonicalized action request
    "nonce": "9f2c...",                    // single-use; enforced by replay protection (EP-7008)
    "requestedAt": "2026-08-06T14:00:00Z"
  },
  "hashChain": {
    "prevProofHash": "sha256:...",         // links to the previous ProofRecord on this boundary
    "thisProofHash": "sha256:..."          // hash over the canonicalized record body
  },
  "signatures": {
    "hmac": "base64...",                   // HMAC integrity layer (fast, symmetric)
    "mldsa65": "base64...",                // ML-DSA-65 post-quantum signature (asymmetric)
    "kmsKeyId": "arn:aws:kms:...",         // AWS KMS key that produced the ML-DSA-65 signature
    "pqPublicKeyId": "pq-2026-08"          // key id resolvable via GET /v2/crypto/pq-public-key
  },
  "issuedAt": "2026-08-06T14:00:00.128Z"
}
```

### 5.2 Dual-layer signing

Each ProofRecord is signed at two independent layers:

1. **HMAC integrity layer** — a fast symmetric MAC over the canonicalized record body. It provides cheap,
   in-band integrity checking and lets the service detect tampering without an asymmetric verification round
   trip.
2. **ML-DSA-65 post-quantum signature** — the record is signed with **hardware-backed ML-DSA-65 post-quantum
   signing via AWS KMS**. The private key never leaves the KMS hardware boundary. Independent verifiers fetch
   the corresponding public key via `GET /v2/crypto/pq-public-key` and verify offline.

> **Conformance note.** The signing scheme follows the RF-100 ProofRecord profile. **Formal RF-100 §8.4
> conformance remains pending independent external review.** We describe the scheme accurately and do not
> claim certified conformance.

### 5.3 Hash chaining

Each ProofRecord embeds the hash of the previous record on the same boundary (`prevProofHash`) and the hash of
its own canonicalized body (`thisProofHash`). This produces an append-only, order-preserving chain per
boundary: a verifier can detect insertion, deletion, or reordering of records, not merely modification of a
single record.

### 5.4 Independent verification

Given a ProofRecord and the boundary's public key, a third party can:

1. Recompute `thisProofHash` over the canonicalized body and compare.
2. Verify the ML-DSA-65 signature against the published public key.
3. Walk `prevProofHash` links to confirm the record's position in the chain.
4. Recompute `actionDigest` from the original action request to confirm the record describes *that* action.

No trust in the caller — or in ExecutionProof at verification time — is required to complete these steps.

## 6. Deployment boundary

A **deployment boundary** (`boundaryId`) is the unit of configuration and enforcement. It scopes:

- which principals have **Authority**,
- which **Evidence** is required,
- the **Constraint** limits (amounts, rates, allow-lists, blast radius),
- the **Control** rules (idempotency window, M-of-N quorum policy),
- the signing keys and the ProofRecord hash chain.

**Payment authorization** (`payment-auth-prod`) is the first production boundary. The **same** engine and
ProofRecord machinery govern other boundaries — e.g. `infra-command-prod` for destructive infrastructure
operations (see [examples/infrastructure-command](examples/infrastructure-command/)). The boundary abstraction
is what makes ExecutionProof general across consequential-action classes rather than payments-specific.

## 7. Threat model

ExecutionProof's job is to make it hard to execute an action that *should not* execute, and to make it
impossible to later deny that a decision was made. The following threats are explicitly in scope.

| Threat | Attack | Mitigation |
|---|---|---|
| **Replay** | Re-submitting a previously-approved request to execute it twice, or replaying a captured ALLOW. | Single-use **nonce** per request, enforced by the Evidence engine; duplicate/expired nonce → **`EP-7008`**. Idempotency keys in the Control engine prevent double-execution of the same intent. |
| **Backdating / forgery of decisions** | Fabricating a ProofRecord, or claiming a decision was made earlier than it was. | Server-issued `issuedAt`, hash-chained records (`prevProofHash`), and ML-DSA-65 signatures. A forged or backdated record fails signature and/or chain verification. |
| **Authority spoofing** | Impersonating an authorized principal to obtain an ALLOW. | Authority engine validates the authenticated principal against the boundary's authority policy; failure → **`EP-7001`**. Authentication is required on every call. |
| **Policy bypass** | Skipping the check, or calling `execute` without a valid verify. | `execute` re-runs verification server-side; an ALLOW cannot be asserted by the caller. Constraint/Control failures → **`EP-7003` / `EP-7004`**. There is no code path that executes without a ProofRecord. |
| **Quorum subversion** | Meeting an M-of-N HOLD with a single approver or an ineligible one. | Distinct-approver and eligibility enforcement in the approval state machine; violation → **`EP-7006`**. |
| **Evidence tampering** | Altering the action after verification. | `actionDigest` binds the ProofRecord to the canonicalized action; a modified action no longer matches its ProofRecord. |

### Out of scope (by design)

- **Delivery of notifications / alerts** and **discovery of who should approve** — ExecutionProof enforces the
  quorum but does not route approvals.
- **Identity provider / SSO integration** — callers authenticate to ExecutionProof and manage their own
  principals; there is no built-in IdP integration yet.
- **Downstream execution correctness** — once ExecutionProof returns ALLOW, whether the rail/control-plane
  executes correctly is the downstream system's responsibility.

These are stated plainly in [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md).

## 8. Error codes

Stable, documented codes returned in the `error.code` field. They map directly to the pipeline.

| Code | Meaning | Engine / stage |
|---|---|---|
| `EP-7001` | Authority invalid, unverified, or not permitted for this boundary. | Authority |
| `EP-7002` | Required evidence missing, malformed, or invalid. | Evidence |
| `EP-7003` | Constraint/limit exceeded (amount, rate, allow-list, blast radius). | Constraint |
| `EP-7004` | Unsafe control state (idempotency conflict, in-flight conflict, quorum not satisfied at execute). | Control |
| `EP-7005` | Boundary not found or misconfigured. | Request validation |
| `EP-7006` | Approval quorum not met, or invalid/duplicate/ineligible approver. | Approval state machine |
| `EP-7007` | ProofRecord not found, or signature/chain verification failed. | ProofRecord retrieval/verification |
| `EP-7008` | Replay detected — nonce reuse or expired nonce. | Evidence (replay protection) |

## 9. Honest production limitations

We keep our claims narrower than our evidence. As of this public release:

- **Notification / alert delivery is not built.** ExecutionProof does not send email, SMS, chat, or pager
  notifications. HOLDs are visible via the API; routing is the caller's responsibility.
- **No identity-provider / SSO integration.** There is no built-in IdP, SSO, or approval-inbox integration.
- **No approval-routing UI.** The approval state machine is API/state-level only; there is no shipped UI for
  reviewing and approving HELD actions.
- **RF-100 §8.4 conformance is not independently certified.** The ProofRecord signing scheme follows the
  RF-100 profile, but **formal RF-100 §8.4 conformance remains pending independent external review.**
- **The four-engine core is documented here; the eight-engine Enterprise pipeline is not part of this repo.**

See [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md) for the full live-vs-not table.

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
