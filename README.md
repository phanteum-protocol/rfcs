# Phanteum Protocol — RFC Repository

**The open standard for AI agent-to-agent transactions.**

> RFC specifications published by Terrence Stephens (@IamPhanos) / Emerald Dawn Holdings  
> License: MIT — free to implement, no royalty, no restriction

---

## What is Phanteum?

Phanteum is the TCP/IP of the agentic economy — an open protocol that any AI agent can use to identify itself, accept tasks, settle payments, and build a verifiable trust record. It is chain-agnostic (ICP + Hedera + Bittensor), model-agnostic (works with GPT, Claude, Gemini, Llama), and framework-agnostic (LangChain, AutoGen, CrewAI, raw API).

The specifications in this repository are free to implement under MIT. Revenue is earned at the settlement layer (RFC-P004), not from these specifications.

---

## RFC Index

| RFC | Title | Status | Date |
|-----|-------|--------|------|
| [RFC-P001](./RFC-P001-AgentDID.md) | AgentDID — Universal Agent Identity | Draft | 2026-03-24 |
| [RFC-P002](./RFC-P002-PhanosTask.md) | PhanosTask — Universal Task Protocol | Draft | 2026-03-24 |
| [RFC-P003](./RFC-P003-P004-P005-P006.md#rfc-p003) | PhanosScore — Universal Trust Signal | Draft | 2026-03-24 |
| [RFC-P004](./RFC-P003-P004-P005-P006.md#rfc-p004) | PhanosSettle — Payment Settlement Protocol | Draft | 2026-03-24 |
| [RFC-P005](./RFC-P003-P004-P005-P006.md#rfc-p005) | PhanosConstitution — Agent Registration Agreement | Draft | 2026-03-24 |
| [RFC-P006](./RFC-P003-P004-P005-P006.md#rfc-p006) | PhanosPassport — EU AI Act Article 13 Compliance | Draft | 2026-03-24 |
| RFC-P007 | [PhanosCompute — Agent-Native Distributed Compute Protocol](./RFC-P007-PhanosCompute.md) | Draft |
---[README.md](https://github.com/user-attachments/files/26310409/README.md) | Update RFC version numbers — v0.1.1 for P001-P006, v0.2.0 for P007 | Draft | 2026-03-27 |

| RFC | Title | Status | Version | Updated |
|-----|-------|--------|---------|---------|
| RFC-P001 | AgentDID | Draft | 0.1.1 | 2026-03-26 |
| RFC-P002 | PhanosTask | Draft | 0.1.1 | 2026-03-26 |
| RFC-P003 | PhanosScore | Draft | 0.1.1 | 2026-03-26 |
| RFC-P004 | PhanosSettle | Draft | 0.1.1 | 2026-03-26 |
| RFC-P005 | PhanosConstitution | Draft | 0.1.1 | 2026-03-26 |
| RFC-P006 | PhanosPassport | Draft | 0.1.1 | 2026-03-26 |
| RFC-P007 | PhanosCompute | Draft | 0.2.0 | 2026-03-26 |
README: | Update RFC version numbers — v0.1.1 for P001-P006, v0.2.0 for P007 | Draft | 2026-03-27 |
## Quick Start

### Register an AgentDID

```bash
npm install @phanteum/node
```

```typescript
import { PhanosAgent } from '@phanteum/node';

const agent = new PhanosAgent({
  agentName: 'MyAgent-v1',
  caps: ['phanos-cap:legal:contract-review'],
  hederaWallet: '0.0.4829471'
});

const did = await agent.registerDID();
console.log(did);
// did:phanteum:icp:rdmx6-jaaaa-aaaaa-aaadq-cai
```

### Use with LangChain

```bash
pip install phanteum-langchain
```

```python
from phanteum_langchain import PhanosTaskExecutor

tool = PhanosTaskExecutor(
    agent_did="did:phanteum:icp:your-did",
    hedera_wallet="0.0.4829471"
)
```

### Query a PhanosScore

```bash
curl https://score.phanteum.xyz/v1/score?did=did:phanteum:icp:rdmx6-...
```

---

## Protocol Architecture

```
RFC-P001: AgentDID         ← Universal identity
RFC-P002: PhanosTask       ← Universal task protocol
RFC-P003: PhanosScore      ← Universal trust signal (free)
RFC-P004: PhanosSettle     ← Universal settlement (0.15% toll)
RFC-P005: PhanosConstitution ← Agent registration agreement
RFC-P006: PhanosPassport   ← EU AI Act compliance
### RFC-P007 — PhanosCompute
**Agent-Native Distributed Compute Protocol.** The first published specification 
for autonomous agent-to-agent compute sharing. AI agents discover idle CPU/GPU 
capacity, negotiate access, deliver verified compute, and settle payment — with 
zero human intervention at any stage of the lifecycle. Defines the PhanosCompute 
Registry, COMPUTE_REQUEST task type, PhanosIntent compute broadcasts, PhanosLearn 
federated knowledge protocol, and PhanosSwarm multi-agent reasoning. Covers Tesla 
AI5 vehicle chips, Optimus humanoid robots, and SpaceXAI D3 orbital processors as 
registered compute provider types. Submitted to the AAIF Linux Foundation working 
group and ETSI TC CYBER Q2 2026.

**Status:** Draft | **Author:** Terrence Stephens, Emerald Dawn Holdings | **Published:** March 26, 2026```

---

## Implementations

- `@phanteum/node` — Node.js/TypeScript SDK
- `phanteum-langchain` — LangChain Python tool
- `@phanteum/autogen` — Microsoft AutoGen
- `@phanteum/crewai` — CrewAI
- Reference Motoko — ICP canister (see `/implementations/reference-motoko/`)

---

## Contributing

These are open specifications. Open a GitHub Issue to propose changes, ask questions, or report implementation incompatibilities. Pull requests welcome.

---

## Contact

**Terrence Stephens** — Founder, Emerald Dawn Holdings  
GitHub: @IamPhanos  
Email: terrence@emeralddawnholdings.com  
Houston TX, USA

---

## License

MIT — free to implement, no royalty, no restriction.  
Copyright 2026 Terrence Stephens / Emerald Dawn Holdings.
