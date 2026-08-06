# ExecutionProof™

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

[![CI](https://github.com/derekhone/executionproof/actions/workflows/ci.yml/badge.svg)](https://github.com/derekhone/executionproof/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![RF-100](https://img.shields.io/badge/Standard-RF--100%20v1.0-blue.svg)](https://github.com/derekhone/rf-100)
[![Website](https://img.shields.io/badge/site-executionproof.io-black.svg)](https://executionproof.io)

**ExecutionProof is an independent pre-execution governance layer for consequential AI actions.**
It sits between an AI agent's *decision* and the *downstream system that would carry it out*, and returns a
signed **ALLOW / HOLD / DENY** verdict — plus a tamper-evident **ProofRecord™** — before anything irreversible happens.

**Verification Before Execution™.**

---

## What this is, in one screen

| Question | Answer |
|---|---|
| **What is it?** | A drop-in verification checkpoint that an AI agent (or its orchestrator) calls *before* executing a consequential action. It answers one question: *should this specific action be allowed to execute right now?* |
| **Who is it for?** | Teams deploying AI agents that can move money, change infrastructure, release data, or grant access — where an incorrect or unauthorized action is expensive or irreversible. |
| **What does it return?** | A signed **ALLOW / HOLD / DENY** decision and a **ProofRecord™** — a tamper-evident receipt of *why* that decision was made, verifiable independently and after the fact. |
| **What is the first production boundary?** | **Payment authorization.** It is the lighthouse deployment, not the whole market — the same architecture governs infrastructure commands, data release, access changes, and deployment operations. |
| **How do I see it work?** | [Five-minute local demo](#five-minute-local-demo) using the shipped example scenarios and the OpenAPI contract. Live hosted API is available under pilot access. |
| **Is it done?** | The decision engine, ProofRecord signing, approval state machine, and replay protection are live. Notification delivery, identity/SSO integration, and an approval-routing UI are **not yet built** — see [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md). We keep this table honest on purpose. |

---

## The problem

AI agents are increasingly trusted to *take actions*, not just produce text: submitting a payment, rotating a
credential, deleting a resource, releasing a dataset. The moment an action is irreversible, "the model was
usually right" is not an acceptable control.

The gap is not intelligence — it is **governance at the point of execution**. Most stacks verify *after* the
fact (logs, monitoring, anomaly detection) or *not at all*. ExecutionProof verifies *before* the action reaches
the system that would execute it, and produces evidence that the check happened and what it decided.

## How it works

ExecutionProof runs a **four-engine sequential pipeline** over every requested action. The engines
short-circuit: the first engine to fail stops the pipeline and produces a **DENY** (or a **HOLD** when the
failure is a policy that requires human approval rather than an outright rejection).

```
                         ┌─────────────────────────────────────────────────────────┐
                         │                    ExecutionProof                        │
                         │           Pre-Execution Governance Layer                 │
   ┌──────────┐          │                                                          │          ┌────────────────┐
   │          │  action  │  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌────────┐  │  verdict │                │
   │ AI Agent ├─────────►│  │ Authority ├─►│ Evidence  ├─►│ Constraint ├─►│ Control │  ├─────────►│  Downstream    │
   │          │  request │  └────┬─────┘  └────┬─────┘  └─────┬──────┘  └───┬────┘  │  + Proof │  System        │
   └──────────┘          │       │ fail        │ fail         │ fail        │ fail  │          │ (payment rail, │
                         │       ▼             ▼              ▼             ▼       │          │  infra, data)  │
                         │      DENY / HOLD (short-circuit on first failure)        │          └────────────────┘
                         │                                                          │
                         │   Verdict ∈ { ALLOW · HOLD · DENY }  +  signed ProofRecord│
                         └─────────────────────────────────────────────────────────┘
```

- **Authority** — Is the requesting agent/principal authorized to perform *this class* of action on *this boundary*?
- **Evidence** — Is the action backed by the required, valid, non-replayed evidence (context, approvals, nonces)?
- **Constraint** — Does the action stay inside the configured limits (amounts, rates, allow-lists, blast radius)?
- **Control** — Is the execution-state safe (idempotency, quorum satisfied, no conflicting in-flight action)?

Each decision emits a **ProofRecord™**: a hash-linked, dual-signed receipt (**hardware-backed ML-DSA-65
post-quantum signing via AWS KMS**, plus an HMAC integrity layer) that can be verified independently, later,
by a party who does not trust the caller. Formal RF-100 §8.4 conformance remains pending independent external
review.

> The **Enterprise** configuration extends this to eight engines (adding Coherence Envelope, Gate Layers, Rail
> Classifier, and Rail Enforcement). This repository documents the four-engine core that is generally available.

See **[ARCHITECTURE.md](ARCHITECTURE.md)** for the full pipeline, ProofRecord format, HOLD / M-of-N approval
flow, and threat model. A rendered version of the diagrams is in **[docs/architecture-diagram.md](docs/architecture-diagram.md)**.

## ALLOW / HOLD / DENY

| Verdict | Meaning | Typical cause |
|---|---|---|
| **ALLOW** | The action passed all four engines. The caller may execute it. The ProofRecord is the evidence it was verified. | Authorized principal, valid evidence, within constraints, safe control state. |
| **HOLD** | The action is not rejected, but may not execute until a required **M-of-N approval quorum** is satisfied via the approval state machine. | High-value payment, sensitive infra change, policy requiring human sign-off. |
| **DENY** | The action must not execute. | Authority failure, missing/invalid/replayed evidence, constraint breach, unsafe control state. |

**HOLD is a state, not an escalation.** ExecutionProof does not itself route to, notify, or "automatically
escalate to" human approvers. It exposes an **M-of-N approval state machine**: approvals are *submitted to* it
via the API, and the HOLD clears only when the quorum is met. Notification/routing is intentionally out of
scope for this layer (see [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md)).

## Five-minute local demo

The demo runs entirely against the shipped example scenarios and the OpenAPI contract — no credentials, no
network, no live keys. It shows the request/response shape and the ProofRecord structure for ALLOW, HOLD, and
DENY on two different boundaries.

```bash
git clone https://github.com/derekhone/executionproof.git
cd executionproof

# 1. Validate the API contract and example payloads (what CI runs)
docker run --rm -v "$PWD":/spec -w /spec python:3.12-slim bash -c "\
  pip install -q pyyaml jsonschema && \
  python -c 'import yaml,glob; [yaml.safe_load(open(f)) for f in [\"docs/openapi.yaml\",\".github/workflows/ci.yml\"]]; print(\"contract + workflow: OK\")' && \
  python -c 'import json,glob; [json.load(open(f)) for f in glob.glob(\"examples/**/scenarios.json\", recursive=True)]; print(\"example scenarios: OK\")'"

# 2. Read the worked examples (request -> verdict -> ProofRecord)
less examples/payment-authorization/scenarios.json
less examples/infrastructure-command/scenarios.json
```

Every request/response pair in `examples/` validates against the schemas in
[`docs/openapi.yaml`](docs/openapi.yaml). To exercise a **live** endpoint (`/v2/boundaries/{boundaryId}/verify`,
`/execute`, `/approve`, and ProofRecord retrieval), request **pilot access** — see
[executionproof.io](https://executionproof.io).

## API at a glance

Full contract: **[docs/openapi.yaml](docs/openapi.yaml)** (OpenAPI 3.0).

| Method & path | Purpose |
|---|---|
| `POST /v2/boundaries/{boundaryId}/verify` | Verify an action → returns ALLOW / HOLD / DENY + ProofRecord. |
| `POST /v2/boundaries/{boundaryId}/execute` | Verify **and** commit execution intent for an ALLOW (idempotent). |
| `POST /v2/boundaries/{boundaryId}/approve` | Submit an approval toward an M-of-N quorum on a HELD action. |
| `GET  /v2/proofrecords/{proofId}` | Retrieve a ProofRecord for independent verification. |
| `GET  /v2/crypto/pq-public-key` | Fetch the current ML-DSA-65 public key to verify ProofRecord signatures. |

Errors use stable codes **EP-7001 … EP-7008** (authority, evidence, constraint, control, boundary, approval
quorum, ProofRecord/signature, and **replay protection**). See [ARCHITECTURE.md](ARCHITECTURE.md#error-codes).

## Worked examples

- **[examples/payment-authorization/](examples/payment-authorization/)** — the first production boundary:
  ALLOW, HOLD (M-of-N), and DENY on a payment authorization request.
- **[examples/infrastructure-command/](examples/infrastructure-command/)** — the *same* engine governing a
  destructive infrastructure command, to show the layer is not payments-only.

## Where this fits

- **Not a payment rail, and not a competitor to one.** Stripe (and equivalents) are **downstream
  execution/rail partners**. ExecutionProof decides *whether* an action should execute; the rail *executes* it.
  ExecutionProof does not move money and does not replace payment rails.
- **Companion tooling.** [VaultProof Agent Guard](https://github.com/derekhone/vaultproof-agent-guard) is a
  free, open-source agent-side guard. [RF-100](https://github.com/derekhone/rf-100) is the governing standard
  (v1.0, public review).
- **Reproducibility.** Preregistered experiments and case records are published for independent scrutiny:
  **253 case records: 250 PASS, 2 FAIL, 1 GATE-STOP** across **75 preregistered experiments**. Datasets are
  archived in the [Remnant Fieldworks Zenodo community](https://zenodo.org/communities/remnant-fieldworks).

## Intellectual property

The methods described here are the subject of **8 non-provisional patent applications pending**. No patents
have been granted; these are pending applications.

## Documentation map

| Document | What's in it |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Four-engine pipeline, ProofRecord signing, ALLOW/HOLD/DENY semantics, M-of-N HOLD flow, deployment boundary, threat model, honest limitations. |
| [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md) | Explicit table of what is live vs. not yet built / pilot-config scope. |
| [docs/openapi.yaml](docs/openapi.yaml) | Full OpenAPI 3.0 contract. |
| [docs/architecture-diagram.md](docs/architecture-diagram.md) | Rendered Mermaid diagrams. |
| [SECURITY.md](SECURITY.md) | Vulnerability reporting, supported versions, response timeline. |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Dev setup, verification harness, reproducing experiments, evidence standards. |
| [CHANGELOG.md](CHANGELOG.md) | Release history. |

## License

Released under the [MIT License](LICENSE). © 2026 Remnant Fieldworks Inc.

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
