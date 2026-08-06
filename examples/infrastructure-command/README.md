# Example: Infrastructure Command

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

**This example exists to show that ExecutionProof is not payments-only.** The *same* four-engine architecture
that governs [payment authorization](../payment-authorization/) governs destructive infrastructure commands.
Only the boundary configuration changes — the pipeline, the ProofRecord machinery, the ALLOW / HOLD / DENY
semantics, and the M-of-N approval state machine are identical.

Every request/response pair here matches the schemas in [`../../docs/openapi.yaml`](../../docs/openapi.yaml).
Full payloads (including ProofRecords) are in [`scenarios.json`](scenarios.json).

> ExecutionProof decides whether a command *may* execute and issues a signed ProofRecord. For an ALLOW, the
> **downstream control plane** (Kubernetes, a cloud API, a Terraform runner, etc.) performs the command.
> ExecutionProof does not run the command itself.

---

## The boundary

`infra-command-prod` is configured with:

- **Authority** — only enrolled SRE/platform agents may issue production infrastructure commands.
- **Evidence** — single-use nonce plus a valid change ticket and open change window.
- **Constraint** — blast radius is bounded; low-impact actions auto-approve, high-impact actions
  (production datastores, namespace deletion) require an M-of-N approval quorum.
- **Control** — idempotency and no conflicting in-flight command against the same target.

## Scenario 1 — ALLOW

An authorized SRE agent restarts a single stateless deployment in a **dev** namespace. Low blast radius, valid
evidence, authorized principal → all four engines pass → **ALLOW** + ProofRecord.

```
verdict: ALLOW
engines: authority PASS -> evidence PASS -> constraint PASS -> control PASS
```

## Scenario 2 — HOLD (M-of-N approval)

A `terraform destroy` targets a **production RDS cluster**. Not rejected, but the Constraint engine flags the
blast radius (`EP-7003`) → **HOLD** with a **2-of-2** quorum. Approvals are submitted to
`POST /v2/boundaries/infra-command-prod/approve`; when both distinct, eligible approvers approve, the HOLD
clears and a new ProofRecord captures the cleared state. ExecutionProof enforces the quorum but does **not**
notify or route to approvers.

```
verify -> HOLD (0/2)
approve(carol) -> HELD (1/2)
approve(dave)  -> CLEARED (2/2) -> eligible to run
```

## Scenario 3 — DENY (authority failure, EP-7001)

An `analytics-bot` — not enrolled to issue production infrastructure commands — attempts to delete the
`payments` namespace. The Authority engine rejects it immediately (`EP-7001`); the pipeline short-circuits and
no later engine runs. Verdict **DENY**, with a ProofRecord recording the rejected decision.

```
verdict: DENY
engines: authority FAIL (EP-7001) -> evidence SKIPPED -> constraint SKIPPED -> control SKIPPED
```

## Verify these examples

```bash
python -c "import json; json.load(open('scenarios.json')); print('scenarios.json: valid JSON')"
```

This is part of what CI runs — see [`.github/workflows/ci.yml`](../../.github/workflows/ci.yml).

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
