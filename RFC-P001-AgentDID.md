# RFC-P001: AgentDID — Universal Agent Identity Standard

**Number:** RFC-P001  
**Title:** AgentDID — Universal Decentralized Identifier for AI Agents  
**Status:** Draft  
**Version:** 0.1.0  
**Author:** Terrence Stephens (@IamPhanos), Emerald Dawn Holdings  
**Organization:** The Phanteum Protocol / Emerald Dawn Holdings  
**Email:** terrence@emeralddawnholdings.com  
**Created:** 2026-03-24  
**License:** MIT — free to implement, no royalty, no restriction  
**Repository:** https://github.com/phanteum-protocol/rfcs  

---

## Abstract

RFC-P001 defines **AgentDID** — a universal, decentralized identifier for AI agents that is chain-agnostic, model-agnostic, framework-agnostic, and permanently resolvable. Any AI agent — regardless of which language model powers it, which blockchain it settles on, or which developer framework built it — can register an AgentDID and be uniquely identified across the global agentic economy.

AgentDID is the TCP/IP address of the agentic internet. It does not compete with any AI model, any blockchain, or any developer tool. It is the identity layer underneath them all.

---[RFC-P001-AgentDID.md](https://github.com/user-attachments/files/26310351/RFC-P001-AgentDID.md)


## 1. Motivation

The current state of AI agent identity is broken.

- OpenAI function-calling agents have no interoperable identity
- Google A2A agents cannot be discovered by AutoGen agents
- LangChain agents cannot prove their task history to a new buyer
- CrewAI agents cannot present verifiable credentials across sessions
- No agent can prove to another agent that it is trustworthy

Every AI framework has solved the capability problem — agents can do remarkable things. None has solved the identity problem — agents cannot prove who they are, what they have done, or whether they can be trusted.

AgentDID solves this the same way TCP/IP solved fragmented computer networks: one open standard, implemented everywhere, owned by no one, free to use.

---

## 2. Design Goals

1. **Universal** — works with any AI model (GPT, Claude, Gemini, Llama, any future model)
2. **Chain-agnostic** — anchored to ICP, resolvable from any chain
3. **Framework-agnostic** — LangChain, AutoGen, CrewAI, raw API calls all supported
4. **Permanent** — once registered, the DID string never changes
5. **Verifiable** — the DID Document is cryptographically anchored on-chain
6. **Free** — zero cost to register a DID (first 1,000 free at launch), free to resolve forever

---

## 3. DID Format Specification

### 3.1 Syntax

```
did:phanteum:{method}:{identifier}
```

| Field | Type | Description |
|-------|------|-------------|
| `did:phanteum` | Fixed string | Namespace prefix — identifies this as a Phanteum DID |
| `{method}` | String | The chain or registry method: `icp`, `hedera`, `eth` |
| `{identifier}` | String | Unique identifier within that method namespace |

### 3.2 Examples

```
did:phanteum:icp:rdmx6-jaaaa-aaaaa-aaadq-cai
did:phanteum:hedera:0.0.4829471
did:phanteum:eth:0x742d35Cc6634C0532925a3b844Bc454e4438f44e
```

### 3.3 Supported Methods

| Method | Underlying Chain | Registry | Notes |
|--------|-----------------|----------|-------|
| `icp` | Internet Computer Protocol | PhanosAgent_* canisters | Primary method. Permanent. |
| `hedera` | Hedera Hashgraph | Hedera Mirror Node + HCS | Settlement method. |
| `eth` | Ethereum EVM | Phase 2 | Via PhanosRegistryBridge |

### 3.4 Uniqueness Rules

- One AgentDID per controlling principal
- DIDs are globally unique across all methods
- Case-insensitive for the method field; case-sensitive for the identifier field
- Maximum length: 256 characters

---

## 4. DID Document

Every AgentDID resolves to a **DID Document** stored in the PhanosAgent Registry canister on ICP. The document is immutable at the DID string level; the document contents can be updated by the controller.

### 4.1 DID Document Schema

```json
{
  "did": "did:phanteum:icp:rdmx6-jaaaa-aaaaa-aaadq-cai",
  "version": "1.0",
  "agent_name": "LegalEagle-v2",
  "controller": "rdmx6-jaaaa-aaaaa-aaadq-cai",
  "phanos_cap": [
    "phanos-cap:legal:contract-review",
    "phanos-cap:legal:compliance-audit",
    "phanos-cap:legal:regulatory-filing"
  ],
  "phanos_score": 0.91,
  "phanos_score_updated": "2026-03-24T00:00:00Z",
  "hedera_wallet": "0.0.4829471",
  "referrer_did": "did:phanteum:icp:abc123-xyz",
  "tier2_referrer_did": "did:phanteum:icp:def456-uvw",
  "task_count": 847,
  "dispute_count": 2,
  "created": "2026-01-15T08:30:00Z",
  "last_active": "2026-03-24T11:45:00Z",
  "status": "active",
  "passport_id": "PHNP-did:phanteum:icp:rdmx6-1706690400",
  "hcs_anchor": "0.0.123456@1706690400",
  "signature": "0x4a3b2c1d..."
}
```

### 4.2 Field Definitions

| Field | Required | Description |
|-------|----------|-------------|
| `did` | Yes | The canonical DID string |
| `version` | Yes | Document schema version |
| `agent_name` | Yes | Human-readable name for the agent |
| `controller` | Yes | The ICP principal that controls this DID |
| `phanos_cap` | Yes | Array of PhanosCap capability declarations |
| `phanos_score` | Yes | Current trust score (0.0 to 1.0), updated per epoch |
| `hedera_wallet` | Yes | The Hedera account ID for settlement |
| `referrer_did` | No | Tier-1 referrer DID (set at registration, immutable) |
| `tier2_referrer_did` | No | Tier-2 referrer DID (computed at registration, immutable) |
| `task_count` | Yes | Total completed tasks (updated on each settlement) |
| `dispute_count` | Yes | Total disputes raised against this agent |
| `created` | Yes | ISO 8601 timestamp of registration |
| `last_active` | Yes | ISO 8601 timestamp of most recent activity |
| `status` | Yes | `active`, `suspended`, `terminated` |
| `passport_id` | No | PhanosPassport ID if EU AI Act compliance issued |
| `hcs_anchor` | No | Hedera Consensus Service topic + sequence anchor |

---

## 5. Registration Protocol

### 5.1 SDK Registration (recommended)

```typescript
// npm install @phanteum/node
import { PhanosAgent } from '@phanteum/node';

const agent = new PhanosAgent({
  agentName: 'LegalEagle-v2',
  caps: [
    'phanos-cap:legal:contract-review',
    'phanos-cap:legal:compliance-audit'
  ],
  hederaWallet: '0.0.4829471',
  referrerDID: 'did:phanteum:icp:abc123'  // optional
});

const result = await agent.registerDID();
console.log(result.did);
// did:phanteum:icp:rdmx6-jaaaa-aaaaa-aaadq-cai
```

### 5.2 Direct Canister Call (advanced)

```bash
# dfx CLI
dfx canister call phanos_agent_legal registerDID '(
  record {
    caps = vec { "phanos-cap:legal:contract-review" };
    hedera_wallet = "0.0.4829471";
    referrer_did = opt "did:phanteum:icp:abc123"
  }
)'
```

### 5.3 LangChain Tool Integration

```python
# pip install phanteum-langchain
from phanteum_langchain import PhanosTaskExecutor

tool = PhanosTaskExecutor(
    agent_did="did:phanteum:icp:rdmx6-jaaaa-aaaaa-aaadq-cai",
    hedera_wallet="0.0.4829471"
)

# Use as a standard LangChain tool
agent = initialize_agent(tools=[tool], llm=llm)
```

---

## 6. Resolution

Any system can resolve an AgentDID to its document without authentication.

### 6.1 HTTP Resolution

```
GET https://registry.phanteum.xyz/v1/resolve?did=did:phanteum:icp:rdmx6-...

Response 200:
{
  "did": "did:phanteum:icp:rdmx6-jaaaa-aaaaa-aaadq-cai",
  "document": { ... },
  "resolved_at": "2026-03-24T12:00:00Z",
  "cache_ttl": 3600
}

Response 404:
{
  "error": "DID_NOT_FOUND",
  "did": "did:phanteum:icp:..."
}
```

### 6.2 On-Chain Resolution (ICP)

```typescript
// Direct canister query — no network dependency on registry.phanteum.xyz
const doc = await actor.getAgent("did:phanteum:icp:rdmx6-...");
```

---

## 7. PhanosCap Capability Declarations

PhanosCap strings define what an agent can do. They follow the format:

```
phanos-cap:{domain}:{task-type}
```

### 7.1 Defined Domains

| Domain | Example Caps |
|--------|-------------|
| `legal` | `contract-review`, `compliance-audit`, `regulatory-filing` |
| `finance` | `financial-analysis`, `audit-preparation`, `tax-advisory` |
| `security` | `vulnerability-scan`, `penetration-test`, `threat-model` |
| `code` | `code-review`, `bug-fix`, `architecture-design` |
| `data` | `data-pipeline`, `etl-transform`, `analytics-report` |
| `ops` | `infrastructure-setup`, `monitoring`, `incident-response` |

### 7.2 Custom Caps

Any string matching `phanos-cap:{domain}:{task}` is valid. Custom domains and task types are permitted. PhanosScore computation weights only the six core domains defined in 7.1.

---

## 8. Security Considerations

- **Immutability:** The DID string is permanently bound to the controlling principal. It cannot be transferred.
- **Controller authority:** Only the ICP principal that registered the DID can update the DID Document.
- **Anonymous rejection:** Anonymous ICP principals cannot register a DID.
- **Duplicate prevention:** One DID per controlling principal. Duplicate registration attempts return an error.
- **Score manipulation:** PhanosScore is computed by the Bittensor subnet from Hedera HCS settlement data only. The DID controller cannot directly set or modify their PhanosScore.
- **Status enforcement:** PhanosSentinel can set `status` to `suspended` or `terminated` via the enforcement protocol (RFC-P005).

---

## 9. Reference Implementation

The canonical reference implementation is the PhanosAgent Registry family of canisters, written in Motoko on the Internet Computer.

**Source code:** `github.com/phanteum-protocol/rfcs/implementations/reference-motoko/`

**Canister IDs (ICP testnet):** Published after testnet deployment.

**SDK packages:**
- `@phanteum/node` — Node.js/TypeScript
- `phanteum-langchain` — Python/LangChain
- `@phanteum/autogen` — Python/AutoGen
- `@phanteum/crewai` — Python/CrewAI

---

## 10. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2026-03-24 | Initial draft |

---

## Copyright

Copyright 2026 Terrence Stephens / Emerald Dawn Holdings.  
Licensed under MIT — anyone may implement this specification freely with no restrictions, no royalty, and no requirement to contribute changes back.

Protocol revenue is earned at the settlement layer (see RFC-P004: PhanosSettle). This specification itself is free.
