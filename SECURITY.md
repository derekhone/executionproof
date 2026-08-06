# Security Policy

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

ExecutionProof is a governance layer for consequential AI actions. We take security reports seriously and
appreciate responsible disclosure.

## Reporting a vulnerability

**Please report security vulnerabilities privately by email to:**

**derek@ownerremnantfieldworks.com**

Use a subject line beginning with `SECURITY:`. Please do **not** open a public GitHub issue for security
vulnerabilities.

Include, where possible:

- A description of the vulnerability and its impact.
- Steps to reproduce (proof-of-concept, request/response samples, affected endpoint or component).
- The version, boundary configuration, or commit affected.
- Any suggested remediation.

If you would like to encrypt your report, request our current PGP key in an initial (non-sensitive) email and
we will provide it.

## Scope

In scope:

- The decision pipeline (Authority, Evidence, Constraint, Control engines) and its short-circuit logic.
- ProofRecord generation, signing (HMAC and ML-DSA-65), hash chaining, and independent verification.
- The M-of-N approval state machine and its distinct-approver / eligibility enforcement.
- Replay protection and nonce/idempotency handling.
- The `/v2` API contract as published in [docs/openapi.yaml](docs/openapi.yaml).
- Authentication and authorization enforcement on the API.

Out of scope:

- Findings that depend on capabilities we already document as **not built** (notification delivery, IdP/SSO
  integration, approval-routing UI) — see [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md).
- Denial-of-service via volumetric traffic against the hosted pilot environment.
- Vulnerabilities in downstream execution/rail partners (e.g. payment rails) rather than in ExecutionProof
  itself.
- Social engineering, physical access, and issues requiring a compromised approver credential that is outside
  the boundary's trust assumptions.

## Supported versions

| Version | Supported |
|---|---|
| `1.0.x` (current public release) | ✅ Security fixes provided |
| `< 1.0` (pre-release / preview) | ❌ Not supported |

We support the latest published minor of the current major release line. When a new minor is released,
security support for the previous minor continues for 90 days.

## Response timeline

We aim to:

| Stage | Target |
|---|---|
| Acknowledge receipt of your report | Within **2 business days** |
| Provide an initial assessment / triage | Within **5 business days** |
| Provide a remediation plan or fix timeline for confirmed issues | Within **15 business days** |
| Coordinated public disclosure | By mutual agreement, after a fix is available |

We will keep you informed throughout and credit reporters who wish to be acknowledged.

## Coordinated disclosure

We ask that you give us a reasonable opportunity to remediate before any public disclosure, and that you avoid
accessing or modifying data that is not yours while investigating. We commit to acting in good faith toward
researchers who do the same, and we will not pursue action against good-faith research consistent with this
policy.

## Contact

Security contact: **derek@ownerremnantfieldworks.com**
Organization: **Remnant Fieldworks Inc.**

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
