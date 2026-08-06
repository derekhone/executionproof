# Changelog

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

All notable changes to this repository are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.0.0-public] — 2026-08-06

First public release of the ExecutionProof product surface: the API contract, architecture, worked examples,
and documentation for the pre-execution governance layer.

### Included

- **README** — product overview, first-screen "what / who / demo / architecture / production status", the
  AI Agent → ExecutionProof → verdict → downstream-system flow diagram, and a five-minute local demo over the
  shipped examples and OpenAPI contract.
- **ARCHITECTURE.md** — the four-engine pipeline (Authority → Evidence → Constraint → Control) with
  short-circuit semantics, ProofRecord dual-layer signing (hardware-backed ML-DSA-65 via AWS KMS + HMAC),
  hash chaining, ALLOW / HOLD / DENY semantics, the M-of-N HOLD approval flow, the deployment-boundary model,
  a threat model (replay, backdating, authority spoofing, policy bypass, quorum subversion, evidence
  tampering), and honest production limitations.
- **PRODUCTION-STATUS.md** — an explicit table of what is live versus not yet built / pilot-config scope.
- **SECURITY.md** — private vulnerability reporting, supported versions, scope, and response timeline.
- **CONTRIBUTING.md** — development setup, the verification harness, reproducing the published experiments,
  evidence standards, and the PR process.
- **docs/openapi.yaml** — OpenAPI 3.0 contract for `verify`, `execute`, `approve`, ProofRecord retrieval, and
  the ML-DSA-65 public-key endpoint, with the ProofRecord schema, the ALLOW / HOLD / DENY enum, and stable
  error codes `EP-7001`…`EP-7008`.
- **docs/architecture-diagram.md** — rendered Mermaid diagrams for the decision pipeline, ProofRecord
  generation, and the HOLD escalation flow.
- **examples/payment-authorization/** — ALLOW / HOLD / DENY worked examples on the first production boundary.
- **examples/infrastructure-command/** — the same engine governing a destructive infrastructure command,
  demonstrating the layer is not payments-only.
- **.github/workflows/ci.yml** — CI: OpenAPI validation, YAML parsing, JSON example validation, Markdown lint,
  and link checking.
- **CITATION.cff**, **LICENSE** (MIT).

### Live in this release

- ALLOW / HOLD / DENY four-engine decision pipeline.
- ProofRecord generation with hardware-backed ML-DSA-65 post-quantum signing via AWS KMS, plus HMAC integrity
  and hash chaining.
- M-of-N approval state machine (API/state level).
- Replay protection (`EP-7008`).
- Payment authorization as the first production deployment boundary.

### Roadmap / not yet built

- Notification & alert delivery (no email/SMS/chat/pager routing).
- Identity-provider / SSO integration and an approval inbox.
- Approval-routing UI.
- Formal RF-100 §8.4 conformance — pending independent external review.
- Eight-engine Enterprise pipeline (Coherence Envelope, Gate Layers, Rail Classifier, Rail Enforcement) —
  available under pilot / enterprise configuration.

See [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md) for the authoritative live-vs-not table.

[1.0.0-public]: https://github.com/derekhone/executionproof/releases/tag/v1.0.0-public

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
