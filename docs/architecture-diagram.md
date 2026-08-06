# Architecture Diagrams

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

Rendered Mermaid diagrams for the ExecutionProof core. See [ARCHITECTURE.md](../ARCHITECTURE.md) for the full
prose description.

---

## 1. Decision pipeline (four engines, short-circuit)

The action flows through the four engines in order. The first engine to fail halts the pipeline and
determines the verdict. Every outcome emits a ProofRecord.

```mermaid
flowchart LR
    A[AI Agent /<br/>Orchestrator] -->|action request| V{{ExecutionProof<br/>boundary}}

    subgraph EP [ExecutionProof pipeline]
        direction LR
        E1[1 · Authority] --> E2[2 · Evidence]
        E2 --> E3[3 · Constraint]
        E3 --> E4[4 · Control]
    end

    V --> E1

    E1 -->|fail EP-7001| D[DENY]
    E2 -->|fail EP-7002 / EP-7008| D
    E3 -->|fail EP-7003| H{HOLD or DENY}
    E4 -->|fail EP-7004| H
    E4 -->|all pass| AL[ALLOW]

    H --> HOLD[HOLD]
    H --> D

    AL --> PR[(ProofRecord<br/>signed + hash-chained)]
    HOLD --> PR
    D --> PR

    AL --> DS[Downstream System<br/>payment rail / infra / data]
    PR -.verify offline.-> AUD[Independent verifier]
```

## 2. ProofRecord generation and signing

Every decision is canonicalized, hash-chained to the previous record on the boundary, then dual-signed
(HMAC + ML-DSA-65 via AWS KMS). Independent verifiers use the published PQ public key.

```mermaid
flowchart TD
    DEC[Verdict + engine results] --> CANON[Canonicalize record body]
    CANON --> DIGEST[Compute actionDigest<br/>sha256 of canonical action]
    CANON --> THIS[Compute thisProofHash]
    PREV[(prevProofHash<br/>last record on boundary)] --> THIS
    THIS --> HMAC[HMAC integrity layer]
    THIS --> KMS[[AWS KMS<br/>hardware-backed<br/>ML-DSA-65]]
    KMS --> SIG[ML-DSA-65 signature]
    HMAC --> REC[ProofRecord]
    SIG --> REC
    DIGEST --> REC
    THIS --> REC
    REC --> STORE[(Append-only<br/>ProofRecord chain)]
    STORE --> NEXTPREV[prevProofHash for next record]

    REC -.GET /v2/proofrecords/id.-> VER[Verifier]
    PUB[[GET /v2/crypto/pq-public-key]] -.-> VER
    VER --> CHECK{Recompute hash +<br/>verify ML-DSA-65 +<br/>walk chain}
```

## 3. HOLD escalation flow (M-of-N approval state machine)

A HOLD parks the action in an M-of-N approval state machine. Approvals are **submitted to** the API;
ExecutionProof enforces the quorum and distinct/eligible approvers but does not notify or route.

```mermaid
stateDiagram-v2
    [*] --> Verifying
    Verifying --> Allowed: all engines pass
    Verifying --> Denied: hard failure (EP-7001/2/3/4/8)
    Verifying --> Held: policy requires M-of-N quorum

    Held --> Held: approve (quorum not yet met)
    Held --> Held: duplicate/ineligible approver (rejected EP-7006)
    Held --> Cleared: M-of-N quorum satisfied
    Cleared --> Allowed: eligible for execute

    Allowed --> [*]
    Denied --> [*]

    note right of Held
        Callers submit approvals via
        POST /v2/boundaries/{id}/approve.
        ExecutionProof does NOT notify or
        route to approvers (see PRODUCTION-STATUS.md).
        Every transition emits a ProofRecord.
    end note
```

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
