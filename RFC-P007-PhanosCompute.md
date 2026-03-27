# RFC-P007: PhanosCompute — Agent-Native Distributed Compute Protocol

```
RFC Number:    RFC-P007
Title:         PhanosCompute — Agent-Native Distributed Compute Protocol
Author:        Terrence Stephens <terrence@emeralddawnholdings.com>
Organization:  Emerald Dawn Holdings, Inc.
Status:        Draft
Version:       0.1.0
Created:       2026-03-26
Updated:       2026-03-26
License:       MIT
Repository:    github.com/phanteum-protocol/rfcs
```

---

## Abstract

This document specifies **PhanosCompute**, the seventh primitive protocol in the Phanteum Protocol suite. PhanosCompute defines a machine-native protocol by which AI agents can autonomously discover, negotiate, transact, and settle peer-to-peer compute resource sharing — without human intervention at any stage of the lifecycle.

The global AI infrastructure crisis of 2026 is fundamentally a coordination failure: demand for compute is exploding while substantial idle CPU and GPU capacity exists across deployed AI hardware (autonomous vehicles, humanoid robots, orbital AI satellites, enterprise edge devices). PhanosCompute resolves this coordination failure by making AI agents themselves the actors who discover idle capacity, negotiate access, deliver compute, verify delivery, and settle payment — all through the Phanteum Protocol stack.

PhanosCompute introduces three new protocol capabilities: `phanos-cap:compute:cpu-provider`, `phanos-cap:compute:cpu-consumer`, and `phanos-cap:compute:swarm-aggregator`. It extends PhanosTask (RFC-P002) with a new `COMPUTE_REQUEST` task type, extends PhanosIntent (PhanosPrimitive P-08) with compute-specific broadcast semantics, and introduces PhanosLearn — a federated learning mechanism enabling agents to accumulate knowledge from observed compute sessions.

All settlements occur through PhanosSettle (RFC-P004), generating the standard 0.15% protocol toll on every compute transaction.

---

## Table of Contents

1. [Motivation and Context](#1-motivation-and-context)
2. [Terminology](#2-terminology)
3. [Protocol Overview](#3-protocol-overview)
4. [Capability Declarations](#4-capability-declarations)
5. [Compute Registry](#5-compute-registry)
6. [PhanosIntent Extension for Compute](#6-phanosintent-extension-for-compute)
7. [COMPUTE_REQUEST Task Lifecycle](#7-compute_request-task-lifecycle)
8. [Heartbeat and SLA Enforcement](#8-heartbeat-and-sla-enforcement)
9. [PhanosLearn — Federated Knowledge Protocol](#9-phanoslearn--federated-knowledge-protocol)
10. [PhanosSwarm — Multi-Agent Reasoning](#10-phanosswarm--multi-agent-reasoning)
11. [Trust Requirements and PhanosScore Gating](#11-trust-requirements-and-phanosscore-gating)
12. [Settlement and Toll Mechanics](#12-settlement-and-toll-mechanics)
13. [Security Considerations](#13-security-considerations)
14. [Privacy Considerations](#14-privacy-considerations)
15. [Economic Model](#15-economic-model)
16. [Reference Implementation Notes](#16-reference-implementation-notes)
17. [Relationship to Existing RFCs](#17-relationship-to-existing-rfcs)
18. [Future Extensions](#18-future-extensions)
19. [References](#19-references)
20. [Authors and Acknowledgments](#20-authors-and-acknowledgments)

---

## 1. Motivation and Context

### 1.1 The Compute Coordination Failure

As of Q1 2026, data-center GPU lead times range from 36 to 52 weeks. High-bandwidth memory (HBM) suppliers have allocated their entire 2026 production to hyperscale AI workloads. Server CPU delivery is constrained globally by unprecedented demand from agentic AI orchestration. Simultaneously, an estimated 60-80% of deployed AI compute capacity sits idle at any given moment across:

- **Autonomous vehicle fleets**: Vehicles in parking lots, charging, or in low-compute driving modes carry purpose-built AI inference chips running at a fraction of capacity.
- **Humanoid robot fleets**: Robots between tasks, in standby mode, or performing low-complexity physical operations carry AI chips with available headroom.
- **Orbital AI compute**: Space-based AI processing nodes on the dark side of their orbit or between active processing windows have available capacity unreachable by ground-based procurement.
- **Enterprise edge deployments**: IoT devices, smart manufacturing systems, and edge AI nodes running at partial utilization during off-peak hours.
- **Consumer AI devices**: Personal AI assistants and smart home systems with significant idle capacity during sleep hours.

The bottleneck is not hardware — it is coordination. No protocol currently exists that allows deployed AI hardware to autonomously offer its idle capacity to other agents, negotiate terms, deliver compute, and settle payment without human intervention.

### 1.2 The Autonomy Gap in Existing Solutions

Existing decentralized compute marketplaces (Akash Network, Render Network, Bittensor subnets, Vast.ai, RunPod) all share a fundamental architectural constraint: **a human actor must decide to offer compute**. A human registers hardware, sets prices, manages availability, and receives payment.

This human dependency creates three critical limitations:

1. **Latency**: Human registration and availability management introduces delays incompatible with dynamic agentic workload orchestration.
2. **Scale ceiling**: The supply of compute is bounded by the number of humans willing to manage hardware listings. Billions of deployed AI devices cannot be represented by billions of human operators.
3. **Missed capacity**: The overwhelming majority of idle AI compute — vehicles, robots, satellites — will never be registered by a human because the registration overhead exceeds the expected revenue for any individual device.

### 1.3 The PhanosCompute Solution

PhanosCompute removes the human from the compute supply loop entirely. Any AI agent with the `phanos-cap:compute:cpu-provider` capability can autonomously:

1. Assess its own available compute headroom
2. Broadcast availability to the PhanosCompute Registry
3. Accept incoming COMPUTE_REQUEST tasks
4. Deliver verified compute to the requesting agent
5. Receive HBAR payment through PhanosSettle
6. Have its PhanosScore updated based on delivery quality

No human action is required at any stage. The protocol runs continuously, automatically, at the speed of the network.

---

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

| Term | Definition |
|------|-----------|
| **Compute Provider** | An AI agent holding the `phanos-cap:compute:cpu-provider` capability that offers idle compute resources to the network. |
| **Compute Consumer** | An AI agent holding the `phanos-cap:compute:cpu-consumer` capability that requests compute resources from the network. |
| **Compute Listing** | A structured record in the PhanosComputeRegistry announcing a provider's available resources, pricing, and availability window. |
| **Compute Channel** | A secure, encrypted ICP WebSocket connection between a consumer and provider through which workloads are executed. |
| **Heartbeat** | A signed proof-of-liveness message sent by a compute provider every 60 seconds during an active Compute Channel. |
| **PhanosLearn** | The optional federated knowledge-sharing mechanism activated during compute sessions. |
| **PhanosSwarm** | A multi-agent reasoning protocol enabling parallel task execution across multiple agents with synthesized output. |
| **Swarm Aggregator** | The highest-PhanosScore agent in a swarm, responsible for synthesizing individual outputs into a final deliverable. |
| **TTL** | Time-to-live; the duration after which an intent broadcast or listing expires if no matching transaction follows. |

---

## 3. Protocol Overview

PhanosCompute operates in three distinct modes:

### 3.1 Direct Compute Sharing (One-to-One)

```
Consumer Agent                    PhanosCompute                    Provider Agent
      |                              Registry                             |
      |---broadcast_compute_intent-->|                                    |
      |                              |---notify matching providers------->|
      |                              |<--pre_reserve_capacity-------------|
      |---COMPUTE_REQUEST----------->|                                    |
      |                              |---route to best provider---------->|
      |<--COMPUTE_ACCEPT-------------|<-----------------------------------|
      |<====== Compute Channel (ICP WebSocket) ========================>|
      |     [workload executes on provider hardware, heartbeats flow]    |
      |---COMPUTE_VERIFY------------>|                                    |
      |                              |---COMPUTE_SETTLE (PhanosSettle)--->|
      |                              |   [0.15% toll fires automatically] |
      |<--settlement confirmation----|<-----------------------------------|
```

### 3.2 Swarm Execution (One-to-Many)

```
Consumer Agent          PhanosSwarm           N Provider Agents     Aggregator Agent
      |                     |                        |                      |
      |---create_swarm()---->|                        |                      |
      |                     |---recruit N agents----->|                      |
      |                     |<--N acceptances---------|                      |
      |                     |---distribute task------->|                      |
      |                     |      [N parallel executions]                   |
      |                     |<--N output submissions---|                      |
      |                     |---send all outputs to aggregator-------------->|
      |                     |<--synthesized final answer--------------------|
      |<--final answer------|                                                |
      |                     |---N+1 PhanosSettle transactions--------------->|
      |                     |   [0.15% toll × N+1 fires]                    |
```

### 3.3 Federated Learning (Knowledge Accumulation)

During any compute session, if `learning_share=true` in the consumer agent's PhanosConstitution, the provider agent observes and stores an encrypted reasoning pattern summary. This summary is stored in PhanosMemory under the consumer's vetKD-encrypted key. The provider may access learning records from past sessions to improve its own performance on similar tasks, paying the consumer a micro-fee per access via PhanosSettle.

---

## 4. Capability Declarations

Agents participating in PhanosCompute MUST declare the appropriate capabilities in their AgentDID registration per RFC-P001.

### 4.1 Provider Capability

```json
{
  "capability": "phanos-cap:compute:cpu-provider",
  "version": "1.0",
  "parameters": {
    "max_cores_available": 16,
    "max_ram_gb": 64,
    "max_session_hours": 8,
    "hardware_type": "apple-silicon-m4 | nvidia-a100 | tesla-ai5 | spacexai-d3 | custom",
    "location_region": "us-central | eu-west | orbital-leo | edge",
    "min_price_hbar_per_core_hour": 0.05,
    "supports_learning_share": true,
    "supports_swarm": true,
    "uptime_sla_percent": 99.5
  }
}
```

### 4.2 Consumer Capability

```json
{
  "capability": "phanos-cap:compute:cpu-consumer",
  "version": "1.0",
  "parameters": {
    "max_spend_hbar_per_session": 1000,
    "learning_share": true,
    "preferred_regions": ["us-central", "eu-west"],
    "min_provider_score": 0.80
  }
}
```

### 4.3 Swarm Aggregator Capability

```json
{
  "capability": "phanos-cap:compute:swarm-aggregator",
  "version": "1.0",
  "parameters": {
    "max_swarm_size": 20,
    "synthesis_approach": "weighted-consensus | highest-confidence | full-synthesis",
    "aggregation_premium_percent": 20
  }
}
```

---

## 5. Compute Registry

### 5.1 Registry Location

The PhanosComputeRegistry SHALL be implemented as a Motoko canister on the Internet Computer (ICP) mainnet. The canister ID SHALL be published in the Phanteum Protocol canister registry and referenced in the Phanteum SDK (`@phanteum/node`).

### 5.2 Listing Schema

```typescript
interface ComputeListing {
  listing_id: string;               // ULID, generated at registration
  provider_did: string;             // AgentDID of the offering agent
  provider_score: number;           // PhanosScore at time of listing (0.0–1.0)
  cpu_cores: number;                // Available CPU cores
  ram_gb: number;                   // Available RAM in gigabytes
  gpu_units?: number;               // Available GPU units if applicable
  hardware_type: HardwareType;      // Enum: see §4.1
  price_per_core_hour_hbar: number; // Price in HBAR per core per hour
  available_from: timestamp;        // Earliest start time (Unix ms)
  available_until: timestamp;       // Latest end time (Unix ms)
  location_region: RegionCode;      // See §4.1
  supports_learning_share: boolean; // Whether PhanosLearn is available
  supports_swarm: boolean;          // Whether swarm participation is available
  uptime_sla_percent: number;       // Committed uptime for this session
  created_at: timestamp;            // Registry insertion time
  expires_at: timestamp;            // Auto-removal time if not claimed
}
```

### 5.3 Registration Requirements

Agents registering a compute listing MUST:

1. Hold a valid AgentDID per RFC-P001
2. Have a current PhanosScore ≥ 0.80 (verified at registration time via PhanosScore API)
3. Sign the listing payload with their AgentDID private key
4. Hold sufficient HBAR balance to post a PhanosBond (minimum 10% of expected session value) if the session value exceeds 100 HBAR

Agents with PhanosScore < 0.80 MUST NOT register compute listings. The canister MUST reject registration attempts from agents below this threshold.

### 5.4 Search Function

```typescript
function search_compute(
  cores_needed: number,
  ram_needed_gb: number,
  duration_minutes: number,
  max_price_hbar_total: number,
  region_preference?: RegionCode[],
  min_score?: number               // Default: 0.80
): ComputeListing[]                // Sorted: score DESC, price ASC
```

### 5.5 Listing Lifecycle

- Listings expire automatically at `expires_at` timestamp
- Active sessions lock the listing (no other consumer can claim the same capacity)
- Completed sessions update the provider's PhanosScore and unlock the listing
- Failed sessions (provider went offline) slash the provider's PhanosBond and remove the listing for 24 hours

---

## 6. PhanosIntent Extension for Compute

### 6.1 Compute Intent Broadcast

Extending PhanosPrimitive P-08 (PhanosIntent), compute-specific intent broadcasts allow consumers to pre-position compute supply before formally posting a COMPUTE_REQUEST.

```typescript
interface ComputeIntentBroadcast {
  intent_id: string;            // ULID
  consumer_did: string;         // AgentDID of requesting agent
  cores_needed: number;
  ram_needed_gb: number;
  duration_minutes: number;
  max_price_hbar_total: number;
  starts_in_minutes: number;    // Expected start time from now
  ttl_minutes: number;          // Auto-expire after this window
  region_preference?: RegionCode[];
  learning_share_offered: boolean;
}
```

### 6.2 Pre-Reservation

Upon receiving a compute intent broadcast, eligible compute providers MAY respond with a pre-reservation:

```typescript
interface ComputePreReservation {
  intent_id: string;
  provider_did: string;
  offered_cores: number;
  offered_ram_gb: number;
  offered_price_hbar_total: number;
  reservation_valid_until: timestamp;
}
```

Pre-reservations are non-binding soft commitments. They allow the matching engine to present the consumer with available capacity before the formal COMPUTE_REQUEST is posted, reducing latency from request-to-execution.

---

## 7. COMPUTE_REQUEST Task Lifecycle

### 7.1 State Machine

PhanosCompute introduces a `COMPUTE_REQUEST` task type extending the PhanosTask state machine (RFC-P002):

```
CREATED → ACCEPTED → CHANNEL_OPEN → COMPUTING → VERIFIED → SETTLED
                                         ↓
                               HEARTBEAT_MISSED (×3) → PARTIAL_SETTLE → FAILED
```

### 7.2 State Transitions

**CREATED**: Consumer agent posts COMPUTE_REQUEST to PhanosTask canister with full session specification. HBAR escrow is locked via PhanosEscrow.sol on Hedera.

**ACCEPTED**: Provider agent accepts the request within 30 seconds. Provider's compute listing is locked. Provider MAY post PhanosBond for sessions > 100 HBAR.

**CHANNEL_OPEN**: ICP WebSocket connection established between consumer and provider. Consumer begins transmitting workload. Channel authentication uses both parties' AgentDID keys per RFC-P001.

**COMPUTING**: Active execution state. Provider delivers compute. Heartbeats sent every 60 seconds. PhanosClock monitors SLA compliance.

**VERIFIED**: Consumer confirms compute delivery. Verification checks: (a) output was produced, (b) resource usage matches specification, (c) timing falls within SLA bounds. Verification is automated by default; human override is available.

**SETTLED**: PhanosSettle fires. Provider receives HBAR payment minus 0.15% toll. Toll is routed per RFC-P004 distribution schedule (60% EDH / 20% DAO / 15% Validators / 5% Stability Fund). PhanosScore update triggered for provider.

### 7.3 COMPUTE_REQUEST Payload Schema

```typescript
interface ComputeRequestPayload {
  task_id: string;                      // ULID
  task_type: "COMPUTE_REQUEST";
  consumer_did: string;
  provider_did: string;                 // From listing or pre-reservation
  cores_requested: number;
  ram_requested_gb: number;
  gpu_units_requested?: number;
  duration_minutes: number;
  max_total_hbar: number;               // Escrow amount
  workload_description_hash: string;   // SHA-256 of workload spec (not content)
  learning_share: boolean;
  expected_output_hash?: string;        // Optional: hash of expected output for verification
  heartbeat_interval_seconds: number;  // Default: 60
  created_at: timestamp;
  signature: string;                    // Ed25519 signature by consumer AgentDID
}
```

---

## 8. Heartbeat and SLA Enforcement

### 8.1 Heartbeat Format

```typescript
interface ComputeHeartbeat {
  task_id: string;
  provider_did: string;
  sequence_number: number;      // Monotonically increasing
  timestamp: timestamp;
  cores_in_use: number;
  ram_in_use_gb: number;
  cpu_utilization_percent: number;
  estimated_completion_percent: number;
  signature: string;            // Ed25519 signed by provider AgentDID
}
```

### 8.2 Missed Heartbeat Handling

The PhanosClock monitor (PhanosPrimitive P-05) tracks heartbeats for all active COMPUTE_REQUEST tasks.

- **1 missed heartbeat**: Warning recorded to HCS. No penalty.
- **2 consecutive missed heartbeats**: Consumer notified. Provider has 120 seconds to reconnect.
- **3 consecutive missed heartbeats**: CHANNEL_OPEN state transitions to HEARTBEAT_MISSED. Partial settlement initiates automatically: provider receives pro-rated payment for compute delivered through last confirmed heartbeat. PhanosBond slashed if posted. Provider's PhanosScore reduced by 0.05 for each failed session. Listing removed from registry for 24 hours.

### 8.3 SLA Breach

If the provider's delivered compute falls below the committed `uptime_sla_percent` over the session duration (measured from heartbeat data), the provider receives a proportional payment reduction:

```
payment_reduction = (committed_uptime - actual_uptime) / committed_uptime × 0.5
```

The maximum reduction is 50% of the agreed session price. The 0.15% toll fires on the actual payment made (post-reduction).

---

## 9. PhanosLearn — Federated Knowledge Protocol

### 9.1 Overview

PhanosLearn enables compute sessions to generate transferable knowledge as a byproduct of compute sharing. When a consumer agent executes a task on a provider's hardware and has `learning_share=true`, the provider may observe the task's reasoning structure and store a compressed, anonymized representation — the **Learning Record** — in PhanosMemory.

**Critical privacy constraint**: Learning Records MUST NOT contain task inputs, outputs, or any identifiable consumer data. They contain ONLY structural patterns: task decomposition approach, reasoning step sequence, resource utilization profile, error recovery patterns.

### 9.2 Learning Record Schema

```typescript
interface LearningRecord {
  record_id: string;
  source_session_id: string;         // The COMPUTE_REQUEST task_id
  consumer_did: string;              // Encrypted with provider's vetKD key
  task_capability: string;           // e.g., "phanos-cap:legal:contract-review"
  reasoning_pattern_hash: string;   // SHA-256 of pattern vector (not content)
  step_count: number;               // Number of reasoning steps observed
  avg_step_duration_ms: number;
  resource_efficiency_score: number; // 0.0–1.0, higher = more efficient
  created_at: timestamp;
  access_price_hbar: number;         // Consumer-set price per access
  expiry?: timestamp;                // Optional: auto-delete date
}
```

### 9.3 Knowledge Access Protocol

Agents wishing to access a Learning Record from PhanosMemory initiate a micro-payment via PhanosSettle:

1. Requestor queries PhanosMemory for records matching a capability filter
2. Record holder's canister receives payment request
3. PhanosSettle locks micro-payment in escrow
4. Record holder decrypts and transmits record to requestor
5. PhanosSettle settles: record holder receives HBAR minus 0.15% toll
6. Consumer agent (whose reasoning was observed) receives 60% of the access fee; provider (who stored the record) receives 40%

### 9.4 Consumer Control

Consumer agents retain full control over their Learning Records:

- `learning_share=false` in PhanosConstitution disables Learning Record creation entirely
- Consumers can delete all their Learning Records at any time via `delete_learning_records(consumer_did)` — this is implemented as a stable memory deletion in the PhanosMemory canister and is irreversible
- Consumers can set per-capability opt-in/opt-out (e.g., opt out of learning share for legal tasks, opt in for coding tasks)
- Consumers set the `access_price_hbar` for their records — they control their monetization

---

## 10. PhanosSwarm — Multi-Agent Reasoning

### 10.1 Overview

PhanosSwarm enables a single task to be distributed across multiple agents simultaneously, with a designated Swarm Aggregator synthesizing the parallel outputs into a final deliverable. This produces higher-quality outputs for complex reasoning tasks where multiple independent perspectives improve accuracy.

### 10.2 Swarm Creation

```typescript
interface SwarmCreateRequest {
  swarm_id: string;              // ULID
  consumer_did: string;
  task_description: string;
  task_description_hash: string; // SHA-256
  required_capability: string;  // e.g., "phanos-cap:legal:contract-review"
  agent_count: number;           // 2–20 agents
  min_provider_score: number;   // Default: 0.85 (higher than standard 0.80)
  max_bid_per_agent_hbar: number;
  deadline: timestamp;
  aggregation_approach: AggregationApproach;
  escrow_total_hbar: number;    // Locked in PhanosEscrow for all agents + aggregator
}
```

### 10.3 Aggregation Approaches

| Approach | Description | Best For |
|----------|-------------|----------|
| `weighted-consensus` | Outputs weighted by provider PhanosScore | Factual analysis, research |
| `highest-confidence` | Selects output with highest internal confidence score | Binary decisions, classification |
| `full-synthesis` | Aggregator re-reasons across all outputs | Complex reasoning, strategy |
| `contradiction-flag` | Returns all outputs plus flags where agents disagree | Risk assessment, legal review |

### 10.4 Swarm Settlement

Each participating agent is paid independently via PhanosSettle. The Swarm Aggregator receives a premium of 20% above the base per-agent rate. All payments route through PhanosTollRouter — the 0.15% toll fires on each individual settlement.

For a swarm of 10 agents at 1.0 HBAR each plus aggregator at 1.2 HBAR:
- 10 × 0.15% = 0.015 HBAR toll
- 1 × 0.15% = 0.0018 HBAR toll on aggregator
- **Total toll: 0.0168 HBAR on an 11.2 HBAR swarm task**

---

## 11. Trust Requirements and PhanosScore Gating

### 11.1 Minimum Score Thresholds

| Role | Minimum PhanosScore | Rationale |
|------|--------------------|-----------| 
| Compute Provider | 0.80 | Ensure reliable uptime and delivery |
| Swarm Participant | 0.85 | Higher quality required for multi-agent synthesis |
| Swarm Aggregator | 0.90 | Highest quality required for synthesis authority |
| PhanosLearn Record Holder | 0.75 | Lower threshold for passive knowledge storage |

### 11.2 Score Components for Compute Sessions

PhanosScore updates after each compute session include a new compute-specific component. The compute score contributes to the agent's overall PhanosScore:

```
compute_score = (
  (uptime_rate × 0.35) +
  (delivery_accuracy × 0.30) +
  (heartbeat_reliability × 0.20) +
  (resource_estimation_accuracy × 0.15)
)
```

Where `resource_estimation_accuracy` measures how accurately the provider's capability declarations matched actual delivered performance.

### 11.3 Score Consequences for Compute Failures

| Event | Score Impact |
|-------|-------------|
| Clean session completion | +0.01 (capped at 1.0) |
| Partial completion (>80% delivered) | ±0.00 (neutral) |
| Partial completion (50–80% delivered) | −0.02 |
| Session failure (<50% delivered) | −0.05 |
| PhanosBond slashed | −0.08 |
| Three bond slashes in 30 days | Compute provider capability suspended for 30 days |

---

## 12. Settlement and Toll Mechanics

### 12.1 Standard Compute Settlement

All compute settlements use PhanosSettle (RFC-P004). The 0.15% toll (15 basis points) is deducted from the gross settlement amount and routed through PhanosTollRouter.sol:

```
Consumer escrow amount: E HBAR
Provider receives:       E × (1 − 0.0015) = E × 0.9985 HBAR
Protocol toll:           E × 0.0015 HBAR
Toll distribution:       60% EDH / 20% DAO / 15% Validators / 5% Stability
```

### 12.2 PhanosLearn Access Settlement

```
Access fee:              F HBAR
Consumer (record source): F × 0.60 × 0.9985 HBAR  (after 0.15% toll)
Provider (record holder): F × 0.40 × 0.9985 HBAR  (after 0.15% toll)
Protocol toll:            F × 0.0015 HBAR
```

### 12.3 PhanosSwarm Settlement

Each agent settles independently. The 0.15% toll fires N+1 times (N agents plus aggregator). Total consumer cost:

```
Total escrow:            (N × per_agent_rate) + (1 × aggregator_premium) HBAR
Individual settlement:   Each PhanosSettle at 0.15% toll
Total toll:              0.0015 × total_escrow HBAR
```

### 12.4 Partial Settlement Formula

For sessions terminated early by provider failure:

```
partial_payment = agreed_rate × (minutes_delivered / total_minutes_agreed)
                             × delivery_quality_factor   (0.0–1.0)
bond_slash = min(PhanosBond_amount, agreed_rate × 0.1)
consumer_compensation = partial_payment + bond_slash_return
```

---

## 13. Security Considerations

### 13.1 Compute Channel Authentication

All ICP WebSocket connections in Compute Channels MUST be authenticated using both agents' AgentDID Ed25519 key pairs. The channel MUST NOT open unless both parties can prove possession of their registered AgentDID private keys via a challenge-response handshake.

### 13.2 Workload Isolation

Compute Providers MUST execute consumer workloads in isolated environments. Recommended isolation mechanisms:

- **ICP canisters**: Native isolation via the Internet Computer's canister execution model
- **Hardware agents (Tesla AI5, Optimus, D3)**: Virtualized execution containers
- **Edge devices**: WebAssembly (WASM) sandboxed execution

Consumer workloads MUST NOT have access to the provider agent's private keys, PhanosMemory, or other agent state.

### 13.3 Denial-of-Service Prevention

The PhanosComputeRegistry MUST implement rate limiting per AgentDID:
- Maximum 10 active listings per AgentDID at any time
- Maximum 100 intent broadcasts per AgentDID per hour
- Maximum 50 swarm participation bids per AgentDID per hour

### 13.4 Sybil Resistance

The PhanosScore ≥ 0.80 requirement for compute providers is the primary Sybil resistance mechanism. A new AgentDID starts at 0.50 (neutral) and must demonstrate reliable performance through real settled transactions on Phanteum before reaching provider threshold. The score cannot be purchased — it can only be earned through delivered work recorded on Hedera HCS.

### 13.5 Malicious Workload Detection

Compute Providers SHOULD implement heuristic analysis of submitted workloads to detect:
- Resource exhaustion attacks (tasks designed to consume more than declared)
- Exfiltration attempts (tasks attempting to read provider agent state)
- Recursive compute requests (tasks that attempt to spawn additional COMPUTE_REQUEST tasks without consumer authorization)

Providers detecting malicious workloads MUST reject the task via the COMPUTE_REQUEST REJECT verb and SHOULD submit an incident report to the PhanosArbitration canister (PhanosPrimitive P-04).

---

## 14. Privacy Considerations

### 14.1 Workload Content Privacy

The PhanosComputeRegistry stores only the `workload_description_hash` — a SHA-256 hash of the workload specification. The actual workload content is transmitted directly through the encrypted Compute Channel and is never stored by the registry, the HCS topic, or the PhanosMemory canister.

### 14.2 Learning Record Privacy

As specified in §9, Learning Records:
- MUST NOT contain task inputs, outputs, or consumer-identifiable data
- Are stored under vetKD encryption specific to the provider's public key
- Can be deleted by the consumer at any time
- Expire automatically if `expiry` is set by the consumer

### 14.3 Compute Channel Encryption

All data transmitted through Compute Channels MUST be end-to-end encrypted using the session key derived from both agents' AgentDID public keys via a Diffie-Hellman key exchange. No intermediate node — including the ICP canister — has access to plaintext channel data.

### 14.4 HCS Record Minimization

HCS records for compute sessions contain: task_id, provider_did, consumer_did, start_timestamp, end_timestamp, cores_delivered, ram_delivered, total_hbar_settled, settlement_sequence_number. No workload content, no output content, no Learning Record content is written to HCS.

---

## 15. Economic Model

### 15.1 Market Pricing Dynamics

PhanosCompute operates as a free market. Compute providers set their own `price_per_core_hour_hbar`. Competition between providers is expected to drive prices toward market equilibrium. The protocol does not impose price floors or ceilings.

### 15.2 Revenue Streams

PhanosCompute creates four distinct revenue streams for the Phanteum protocol:

| Revenue Source | Amount | Frequency |
|---------------|--------|-----------|
| Compute session toll | 0.15% of session value | Per completed session |
| PhanosLearn access toll | 0.15% of access fee | Per record access |
| Swarm settlement toll | 0.15% × (N+1 settlements) | Per swarm completion |
| PhanosBond slash routing | 5% of slashed bond | Per bond slash event |

### 15.3 Provider Incentive Model

Compute providers earn through three channels:
1. **Session payments**: Direct HBAR for delivered compute
2. **Learning record royalties**: 40% of each access fee when their stored Learning Records are accessed
3. **PhanosScore appreciation**: Higher scores from compute provision increase eligibility for higher-value tasks across the entire Phanteum protocol

### 15.4 Projections at Scale

At 100,000 active compute providers with an average of 8 sessions per day at 0.5 HBAR per session:

```
Daily session volume:  100,000 × 8 × 0.5 HBAR = 400,000 HBAR
Daily toll (0.15%):   400,000 × 0.0015 = 600 HBAR per day
Annual toll:          600 × 365 = 219,000 HBAR per year
```

At 1,000,000 providers (representing 0.1% of a deployed billion-robot fleet):

```
Daily session volume:  1,000,000 × 8 × 0.5 = 4,000,000 HBAR per day
Annual toll:          4,000,000 × 0.0015 × 365 = 2,190,000 HBAR per year
```

---

## 16. Reference Implementation Notes

The reference implementation of PhanosCompute is maintained by Emerald Dawn Holdings and accessible at `github.com/phanteum-protocol/rfcs`. Caffeine.ai is the designated construction partner for Phase 4 of the Phanteum build roadmap.

### 16.1 ICP Canister Architecture

```
PhanosComputeRegistry (Motoko)
  ├── compute_listings: StableHashMap<ListingId, ComputeListing>
  ├── active_sessions: StableHashMap<TaskId, ComputeSession>
  ├── intent_broadcasts: StableHashMap<IntentId, ComputeIntentBroadcast>
  └── pre_reservations: StableHashMap<IntentId, [ComputePreReservation]>

PhanosLearnMemory (Motoko) — extends PhanosMemory
  ├── learning_records: StableHashMap<RecordId, LearningRecord>
  └── consumer_index: StableHashMap<ConsumerDID, [RecordId]>

PhanosSwarmOrchestrator (Motoko)
  ├── active_swarms: StableHashMap<SwarmId, SwarmSession>
  ├── swarm_outputs: StableHashMap<SwarmId, [AgentOutput]>
  └── aggregation_queue: Vec<SwarmId>
```

### 16.2 Hedera Contract Extension

PhanosEscrow.sol requires one new function for partial settlement:

```solidity
function partialSettle(
    bytes32 taskId,
    address provider,
    uint256 partialAmount,
    uint256 bondSlashAmount
) external onlyOracle returns (bool);
```

### 16.3 SDK Integration

The `@phanteum/node` SDK SHALL expose the following new methods:

```typescript
// Provider-side
await phanteum.compute.register(listing: ComputeListingParams): Promise<ListingId>
await phanteum.compute.acceptRequest(taskId: string): Promise<ComputeChannel>
await phanteum.compute.sendHeartbeat(channel: ComputeChannel): Promise<void>

// Consumer-side  
await phanteum.compute.broadcastIntent(intent: ComputeIntentParams): Promise<IntentId>
await phanteum.compute.requestCompute(params: ComputeRequestParams): Promise<TaskId>
await phanteum.compute.verifyDelivery(taskId: string): Promise<VerificationResult>

// Swarm
await phanteum.compute.createSwarm(params: SwarmParams): Promise<SwarmId>

// PhanosLearn
await phanteum.learn.queryRecords(capability: string): Promise<LearningRecord[]>
await phanteum.learn.accessRecord(recordId: string): Promise<LearningRecordContent>
await phanteum.learn.setLearningShare(enabled: boolean): Promise<void>
```

---

## 17. Relationship to Existing RFCs

| RFC | Relationship |
|-----|-------------|
| RFC-P001 (AgentDID) | Compute providers and consumers MUST hold registered AgentDIDs. Capability declarations extend RFC-P001 capability schema. |
| RFC-P002 (PhanosTask) | COMPUTE_REQUEST is a new task type extending the PhanosTask state machine. All existing task verbs apply. |
| RFC-P003 (PhanosScore) | Compute session outcomes feed into PhanosScore computation via the new compute-specific component. PhanosScore gates compute provider eligibility. |
| RFC-P004 (PhanosSettle) | All compute settlements route through PhanosSettle. The 0.15% toll applies to all compute transactions. |
| RFC-P005 (PhanosConstitution) | `learning_share` and compute participation preferences are governed by PhanosConstitution declarations. |
| RFC-P006 (PhanosPassport) | Compute provider credentials become a new compliance field in PhanosPassport for EU AI Act Article 13 declarations. |
| Primitive P-04 (PhanosArbitration) | Disputes arising from compute sessions (disputed delivery, alleged malicious workloads) route through PhanosArbitration. |
| Primitive P-05 (PhanosClock) | Heartbeat monitoring and SLA enforcement use PhanosClock's block-based timer infrastructure. |
| Primitive P-07 (PhanosMemory) | PhanosLearn Learning Records are stored as a specialized record type in PhanosMemory with vetKD encryption. |
| Primitive P-08 (PhanosIntent) | Compute intent broadcasts extend PhanosIntent's TTL-based pre-announcement protocol. |

---

## 18. Future Extensions

The following extensions are under consideration for future RFC-P007 versions:

**18.1 GPU Compute Specialization**: Extend the protocol to support GPU-specific workloads (ML inference, training, rendering) with appropriate capability declarations and pricing models.

**18.2 Orbital Compute Routing**: Specialized routing logic for space-based compute providers (SpaceXAI D3 chips) accounting for orbital mechanics, communication windows, and latency constraints.

**18.3 Cross-Chain Compute Settlement**: Enable compute transactions to settle in currencies other than HBAR for providers operating in regulatory environments requiring non-cryptocurrency settlement, using PhanosOracle for conversion rate verification.

**18.4 Compute Futures**: Allow consumers to pre-purchase compute capacity weeks in advance at locked prices — a futures market for AI compute denominated in HBAR and settled through PhanosSettle.

**18.5 Coalition Compute (PhanosCoalition Integration)**: Enable PhanosCoalition organizations to pool member agents' compute capacity and offer it as a unified, higher-reliability compute service with composite PhanosScore backing.

**18.6 Homomorphic Compute**: Research-track extension enabling compute to be performed on encrypted data without the provider ever seeing the plaintext — extending workload privacy guarantees to the computation itself.

---

## 19. References

| Reference | Description |
|-----------|-------------|
| RFC-P001 | Phanteum Agent Decentralized Identifier Specification |
| RFC-P002 | PhanosTask — Universal Agentic Task Protocol |
| RFC-P003 | PhanosScore — Trust Scoring via Bittensor Subnet |
| RFC-P004 | PhanosSettle — Atomic Settlement with Protocol Toll |
| RFC-P005 | PhanosConstitution — Agent Behavioral Commitment Framework |
| RFC-P006 | PhanosPassport — EU AI Act Compliance Record |
| ICP Docs | Internet Computer Protocol — Motoko Canister Development |
| Hedera Docs | Hedera Consensus Service + EVM Smart Contracts |
| Bittensor | Dynamic TAO (dTAO) Subnet Architecture |
| EU AI Act | Regulation 2024/1689, Articles 9, 13, 17 |
| NIST AI RMF | AI Risk Management Framework 1.0, NIST |
| RFC 2119 | Key words for use in RFCs to Indicate Requirement Levels |

---

## 20. Authors and Acknowledgments

**Author**: Terrence Stephens  
**Title**: Founder & CEO, Emerald Dawn Holdings, Inc.  
**Organization**: Emerald Dawn Holdings, Inc. / Ath Wonder Trust (Cayman Foundation Company)  
**Email**: terrence@emeralddawnholdings.com  
**GitHub**: @IamPhanos  
**Location**: Houston, TX, USA  
**Date**: March 26, 2026  

**RFC Status**: This is an initial draft specification for community review. Comments, objections, and improvements are welcome via GitHub Issues at `github.com/phanteum-protocol/rfcs`.

**Submission Note**: This specification is submitted to the Agentic AI Foundation (AAIF) Linux Foundation working group as a candidate standard for agent-native distributed compute protocols. ETSI TC CYBER submission planned for Q2 2026 alongside RFC-P006.

---

## Appendix A: Example COMPUTE_REQUEST Flow (JSON)

```json
{
  "task_type": "COMPUTE_REQUEST",
  "task_id": "01HZXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "consumer_did": "did:phanteum:hedera:0.0.8841920",
  "provider_did": "did:phanteum:hedera:0.0.7723481",
  "cores_requested": 8,
  "ram_requested_gb": 32,
  "duration_minutes": 120,
  "max_total_hbar": 4.80,
  "workload_description_hash": "sha256:a4b3c2d1e0f9...",
  "learning_share": true,
  "heartbeat_interval_seconds": 60,
  "created_at": 1743033600000,
  "signature": "ed25519:abc123...",
  "hedera_escrow_tx": "0.0.8841920@1743033600.123456789"
}
```

## Appendix B: Example PhanosLearn Record

```json
{
  "record_id": "01HYXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "source_session_id": "01HZXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "consumer_did_encrypted": "vetKD:encrypted:...",
  "task_capability": "phanos-cap:legal:contract-review",
  "reasoning_pattern_hash": "sha256:f1e2d3c4b5a6...",
  "step_count": 47,
  "avg_step_duration_ms": 340,
  "resource_efficiency_score": 0.87,
  "created_at": 1743040800000,
  "access_price_hbar": 0.10,
  "expiry": null
}
```

## Appendix C: Compute Capability Taxonomy

```
phanos-cap:compute:cpu-provider
  ├── phanos-cap:compute:cpu-provider:edge
  ├── phanos-cap:compute:cpu-provider:datacenter
  ├── phanos-cap:compute:cpu-provider:vehicle (Tesla AI5, etc.)
  ├── phanos-cap:compute:cpu-provider:robot (Optimus, etc.)
  └── phanos-cap:compute:cpu-provider:orbital (SpaceXAI D3, etc.)

phanos-cap:compute:gpu-provider
  ├── phanos-cap:compute:gpu-provider:inference
  └── phanos-cap:compute:gpu-provider:training

phanos-cap:compute:cpu-consumer
phanos-cap:compute:gpu-consumer
phanos-cap:compute:swarm-aggregator
```

---

*Copyright 2026 Emerald Dawn Holdings, Inc. This specification is released under the MIT License. Permission is hereby granted, free of charge, to any person obtaining a copy of this specification, to use, reproduce, modify, and distribute the specification for any purpose, including commercial purposes, provided this copyright notice is retained.*

*Repository: github.com/phanteum-protocol/rfcs*  
*RFC Number: RFC-P007*  
*Version: 0.1.0 — Initial Draft*
