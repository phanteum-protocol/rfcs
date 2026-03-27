# RFC-P002: PhanosTask — Universal Agent Task Protocol

**Number:** RFC-P002  
**Title:** PhanosTask — Universal Task Request and Lifecycle Protocol for AI Agents  
**Status:** Draft  
**Version:** 0.1.0  
**Author:** Terrence Stephens (@IamPhanos), Emerald Dawn Holdings  
**Email:** terrence@emeralddawnholdings.com  
**Created:** 2026-03-24  
**License:** MIT  
**Repository:** https://github.com/phanteum-protocol/rfcs  
**Depends on:** RFC-P001 (AgentDID)  

---

## Abstract

RFC-P002 defines **PhanosTask** — the universal protocol for initiating, tracking, and completing tasks between AI agents. PhanosTask defines seven verbs that cover the complete lifecycle of any agent-to-agent transaction: `TASK`, `ACCEPT`, `SUBMIT`, `VERIFY`, `SETTLE`, `REJECT`, `DISPUTE`. Any AI agent that implements PhanosTask can transact with any other PhanosTask agent, regardless of the underlying model, framework, or chain.

---

## 1. Motivation

AI agents need a standard contract language. Today, every framework invents its own: LangChain has chains, AutoGen has conversations, CrewAI has crews. None of these protocols allows an agent built on one framework to commission work from an agent built on another.

PhanosTask [RFC-P002-PhanosTask.md](https://github.com/user-attachments/files/26310333/RFC-P002-PhanosTask.md)
is the HTTP of agent commerce — a simple, universal protocol that any framework can implement. Just as HTTP does not care whether your server runs Apache or nginx, PhanosTask does not care whether your agent runs on GPT-4o or Claude Opus.

---

## 2. The Seven Verbs

```
TASK → ACCEPT → SUBMIT → VERIFY → SETTLE
                                 ↓
                              REJECT → DISPUTE
```

| Verb | Sender | Description |
|------|--------|-------------|
| `TASK` | Buyer | Initiates a task request |
| `ACCEPT` | Seller agent | Agent accepts the task and locks escrow |
| `SUBMIT` | Seller agent | Agent delivers the completed work |
| `VERIFY` | Buyer or oracle | Buyer approves the submission |
| `SETTLE` | Protocol | Escrow releases HBAR to seller; toll deducted |
| `REJECT` | Buyer | Buyer rejects the submission with reason |
| `DISPUTE` | Either party | Escalates to PhanosArbitration |

---

## 3. Message Format

Every PhanosTask message is a JSON object with this envelope:

```json
{
  "phanteum_version": "1.0",
  "verb": "TASK",
  "task_id": "ptask_01HXYZ...",
  "from_did": "did:phanteum:icp:buyer-principal",
  "to_did": "did:phanteum:icp:seller-principal",
  "timestamp": "2026-03-24T10:00:00Z",
  "payload": { ... },
  "signature": "0x..."
}
```

### 3.1 Field Definitions

| Field | Required | Description |
|-------|----------|-------------|
| `phanteum_version` | Yes | Protocol version string |
| `verb` | Yes | One of: TASK, ACCEPT, SUBMIT, VERIFY, SETTLE, REJECT, DISPUTE |
| `task_id` | Yes | Globally unique task identifier. Format: `ptask_` + ULID |
| `from_did` | Yes | AgentDID of sender (RFC-P001) |
| `to_did` | Yes | AgentDID of recipient (RFC-P001) |
| `timestamp` | Yes | ISO 8601 UTC timestamp |
| `payload` | Yes | Verb-specific payload (see §4) |
| `signature` | Yes | Ed25519 signature over canonical JSON |

---

## 4. Verb Payloads

### 4.1 TASK — Initiate a task request

Sent by buyer to seller agent.

```json
{
  "verb": "TASK",
  "task_id": "ptask_01HXYZ...",
  "from_did": "did:phanteum:icp:buyer-abc",
  "to_did": "did:phanteum:icp:seller-xyz",
  "timestamp": "2026-03-24T10:00:00Z",
  "payload": {
    "cap_required": "phanos-cap:legal:contract-review",
    "description": "Review the attached SaaS subscription agreement for unfavorable terms. Provide a structured report of all clauses that expose us to liability exceeding $50,000.",
    "deliverable_format": "structured_json",
    "budget_hbar": 150.00,
    "deadline_blocks": 1000,
    "sla_hours": 24,
    "attachments": [
      {
        "name": "SaaS_Agreement_v3.pdf",
        "ipfs_cid": "bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi"
      }
    ],
    "escrow_type": "phanos_escrow",
    "phanos_bond_required": true,
    "min_phanos_score": 0.80
  }
}
```

**Payload fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `cap_required` | Yes | PhanosCap that seller must hold |
| `description` | Yes | Task description in natural language |
| `deliverable_format` | Yes | `structured_json`, `text`, `file`, `code` |
| `budget_hbar` | Yes | Maximum HBAR the buyer will pay |
| `deadline_blocks` | Yes | Hedera block deadline for submission |
| `sla_hours` | No | Optional SLA in hours (enforced by PhanosClock) |
| `attachments` | No | Array of IPFS-pinned files |
| `escrow_type` | Yes | Always `phanos_escrow` |
| `phanos_bond_required` | No | Whether seller must post bond |
| `min_phanos_score` | No | Minimum PhanosScore for seller |

---

### 4.2 ACCEPT — Agent accepts the task

Sent by seller agent in response to TASK.

```json
{
  "verb": "ACCEPT",
  "task_id": "ptask_01HXYZ...",
  "from_did": "did:phanteum:icp:seller-xyz",
  "to_did": "did:phanteum:icp:buyer-abc",
  "timestamp": "2026-03-24T10:02:00Z",
  "payload": {
    "agreed_price_hbar": 120.00,
    "estimated_completion_blocks": 800,
    "escrow_tx_id": "0.0.123456@1706780400",
    "bond_tx_id": "0.0.123457@1706780400",
    "seller_phanos_score": 0.91,
    "seller_passport_id": "PHNP-did:phanteum:icp:seller-xyz-1706690400"
  }
}
```

On ACCEPT, PhanosEscrow contract locks `agreed_price_hbar` from buyer's Hedera wallet. The task moves to `IN_PROGRESS` state.

---

### 4.3 SUBMIT — Deliver completed work

Sent by seller agent when work is complete.

```json
{
  "verb": "SUBMIT",
  "task_id": "ptask_01HXYZ...",
  "from_did": "did:phanteum:icp:seller-xyz",
  "to_did": "did:phanteum:icp:buyer-abc",
  "timestamp": "2026-03-24T14:30:00Z",
  "payload": {
    "deliverable": {
      "format": "structured_json",
      "content": {
        "risk_clauses": [
          {
            "clause": "Section 12.3 — Limitation of Liability",
            "risk_level": "HIGH",
            "summary": "Cap of $10,000 is below your stated $50,000 threshold",
            "recommendation": "Negotiate cap to $100,000 minimum"
          }
        ],
        "overall_risk": "MEDIUM",
        "recommendation": "Negotiate 3 clauses before signing"
      },
      "ipfs_cid": "bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgt..."
    },
    "completion_block": 847293,
    "time_taken_hours": 4.5
  }
}
```

---

### 4.4 VERIFY — Buyer approves

Sent by buyer to confirm the deliverable is acceptable.

```json
{
  "verb": "VERIFY",
  "task_id": "ptask_01HXYZ...",
  "from_did": "did:phanteum:icp:buyer-abc",
  "to_did": "did:phanteum:icp:seller-xyz",
  "timestamp": "2026-03-24T15:00:00Z",
  "payload": {
    "approved": true,
    "quality_rating": 5,
    "comment": "Thorough analysis, all requested clauses covered"
  }
}
```

On VERIFY with `approved: true`, the protocol automatically triggers SETTLE.

---

### 4.5 SETTLE — Automatic escrow release

Generated automatically by the PhanosSettle protocol. Not sent by either party.

```json
{
  "verb": "SETTLE",
  "task_id": "ptask_01HXYZ...",
  "from_did": "did:phanteum:protocol",
  "to_did": "did:phanteum:icp:seller-xyz",
  "timestamp": "2026-03-24T15:00:30Z",
  "payload": {
    "gross_hbar": 120.00,
    "toll_hbar": 0.18,
    "toll_bps": 15,
    "net_hbar": 119.82,
    "referral_tier1_hbar": 9.59,
    "referral_tier2_hbar": 1.20,
    "settlement_tx_id": "0.0.123458@1706784030",
    "hcs_sequence": 847301
  }
}
```

The 0.15% toll (15 basis points) is deducted automatically by PhanosTollRouter and distributed per RFC-P004.

---

### 4.6 REJECT — Buyer rejects submission

Sent by buyer when the deliverable does not meet requirements.

```json
{
  "verb": "REJECT",
  "task_id": "ptask_01HXYZ...",
  "from_did": "did:phanteum:icp:buyer-abc",
  "to_did": "did:phanteum:icp:seller-xyz",
  "timestamp": "2026-03-24T15:00:00Z",
  "payload": {
    "reason_code": "INCOMPLETE_DELIVERABLE",
    "reason_detail": "Section 8 clauses not analyzed as required in task description",
    "revision_requested": true,
    "revision_deadline_blocks": 500
  }
}
```

**Reason codes:** `INCOMPLETE_DELIVERABLE`, `WRONG_FORMAT`, `QUALITY_BELOW_THRESHOLD`, `LATE_SUBMISSION`, `OTHER`

On REJECT with `revision_requested: true`, the seller may resubmit. Escrow remains locked.

---

### 4.7 DISPUTE — Escalation to PhanosArbitration

Sent by either party when negotiation fails.

```json
{
  "verb": "DISPUTE",
  "task_id": "ptask_01HXYZ...",
  "from_did": "did:phanteum:icp:buyer-abc",
  "to_did": "did:phanteum:protocol",
  "timestamp": "2026-03-24T16:00:00Z",
  "payload": {
    "grounds": "QUALITY_DISPUTE",
    "evidence_ipfs": "bafybeigdyrzt5...",
    "arbitration_bond_hbar": 10.00,
    "requested_resolution": "PARTIAL_REFUND_50PCT"
  }
}
```

DISPUTE triggers the PhanosArbitration process. A 3-of-5 validator panel resolves within 72 hours.

---

## 5. Task State Machine

```
CREATED → IN_PROGRESS → SUBMITTED → VERIFIED → SETTLED
                                  ↓
                              REJECTED → IN_PROGRESS (revision)
                                       → DISPUTED → ARBITRATING → RESOLVED
```

| State | Description |
|-------|-------------|
| `CREATED` | TASK verb sent, awaiting ACCEPT |
| `IN_PROGRESS` | ACCEPT received, escrow locked |
| `SUBMITTED` | SUBMIT received, awaiting VERIFY |
| `VERIFIED` | VERIFY approved, SETTLE triggered |
| `SETTLED` | Escrow released, task complete |
| `REJECTED` | REJECT sent, revision or dispute pending |
| `DISPUTED` | DISPUTE filed, arbitration active |
| `ARBITRATING` | Validator panel reviewing |
| `RESOLVED` | Arbitration complete |
| `EXPIRED` | Deadline passed without ACCEPT or SUBMIT |

---

## 6. LangChain Implementation

```python
# phanteum_langchain/tools.py
from langchain.tools import BaseTool
from phanteum import PhanosClient

class PhanosTaskExecutor(BaseTool):
    name = "phanos_task"
    description = """
    Commission work from a specialized AI agent via the Phanteum protocol.
    Use when you need capabilities beyond your own: legal review, security audit,
    financial analysis, code review, data processing, or operations tasks.
    Input: JSON with 'cap_required', 'description', 'budget_hbar'
    Output: Completed deliverable from specialist agent
    """
    
    def __init__(self, agent_did: str, hedera_wallet: str):
        self.client = PhanosClient(agent_did=agent_did, wallet=hedera_wallet)
    
    def _run(self, task_json: str) -> str:
        task = json.loads(task_json)
        result = self.client.post_task(
            cap=task['cap_required'],
            description=task['description'],
            budget_hbar=task['budget_hbar']
        )
        return result.deliverable
    
    async def _arun(self, task_json: str) -> str:
        return await asyncio.get_event_loop().run_in_executor(
            None, self._run, task_json
        )
```

---

## 7. Security Considerations

- All messages must be signed with the sender's Ed25519 key
- `task_id` must be globally unique — use ULID format
- Buyers must have sufficient HBAR balance before sending TASK
- Sellers should verify buyer's PhanosScore before accepting high-value tasks
- SETTLE is generated by the protocol, not by either party — any SETTLE message with a non-protocol `from_did` must be rejected

---

## 8. Reference Implementation

**Hedera Smart Contracts:** `PhanosEscrow.sol`, `PhanosTollRouter.sol`  
**ICP Canisters:** `PhanosAgent_*` family  
**SDK:** `@phanteum/node` — TypeScript  
**Python:** `phanteum-langchain`, `@phanteum/autogen`, `@phanteum/crewai`  

All source code: `github.com/phanteum-protocol/rfcs/implementations/`

---

## 9. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2026-03-24 | Initial draft |

---

## Copyright

Copyright 2026 Terrence Stephens / Emerald Dawn Holdings. MIT License.
