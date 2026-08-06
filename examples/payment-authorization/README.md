# Example: Payment Authorization

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

**Payment authorization is the first production deployment boundary for ExecutionProof.** It is the lighthouse
deployment — not the whole market. The same four-engine architecture governs infrastructure commands, data
release, access changes, and other consequential actions (see the
[infrastructure-command example](../infrastructure-command/)).

This example walks through three verdicts — **ALLOW**, **HOLD**, and **DENY** — on the `payment-auth-prod`
boundary. Every request/response pair here matches the schemas in [`../../docs/openapi.yaml`](../../docs/openapi.yaml).
The full payloads (including ProofRecords) are in [`scenarios.json`](scenarios.json).

> **ExecutionProof does not move money.** It decides whether a payment may be authorized and produces a
> signed ProofRecord. For an ALLOW, a **downstream execution/rail partner (e.g. Stripe)** performs the actual
> transfer. Stripe and equivalents are downstream partners, not competitors — ExecutionProof does not replace
> payment rails.

---

## The boundary

`payment-auth-prod` is configured with:

- **Authority** — only enrolled billing agents may authorize payments.
- **Evidence** — a single-use nonce and a verified-invoice context are required.
- **Constraint** — payee must be allow-listed; amounts at or below the auto-approve threshold pass, amounts
  above it require an M-of-N approval quorum.
- **Control** — idempotency and no conflicting in-flight authorization for the same reference.

## Scenario 1 — ALLOW

An authorized billing agent authorizes a small, well-evidenced invoice payment ($42.00). All four engines
pass; the verdict is **ALLOW** and a ProofRecord is issued. The caller may forward the authorized payment to
the downstream rail.

```
verdict: ALLOW
engines: authority PASS -> evidence PASS -> constraint PASS -> control PASS
proof:   ep_pr_01J9ZK7QW3H8X2M4N6P0R5T7VB
```

## Scenario 2 — HOLD (M-of-N approval)

A large payment ($250,000.00) exceeds the auto-approve threshold. It is **not rejected** — the Constraint
engine flags it (`EP-7003`) and the verdict is **HOLD** with a **2-of-3** quorum required.

**HOLD is a state, not an escalation.** ExecutionProof does not notify or route to approvers. Callers submit
approvals to `POST /v2/boundaries/payment-auth-prod/approve`. When two distinct, eligible approvers have
approved, the quorum is satisfied, the HOLD clears, and a new ProofRecord captures the cleared state.

```
verify -> HOLD (0/2)
approve(alice)  -> HELD (1/2)
approve(bob)    -> CLEARED (2/2) -> eligible for execute
approve(alice again) -> rejected EP-7006 (distinct-approver rule)
```

## Scenario 3 — DENY (replay protection, EP-7008)

A previously-used nonce is re-submitted. The Evidence engine's **replay protection** rejects it with
`EP-7008` before any constraint or control check runs — the pipeline short-circuits. Verdict **DENY**. A
ProofRecord is still issued: a denied action is a verified decision with evidence.

```
verdict: DENY
engines: authority PASS -> evidence FAIL (EP-7008) -> constraint SKIPPED -> control SKIPPED
```

## Verify these examples

```bash
python -c "import json; json.load(open('scenarios.json')); print('scenarios.json: valid JSON')"
```

This is part of what CI runs — see [`.github/workflows/ci.yml`](../../.github/workflows/ci.yml).

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
