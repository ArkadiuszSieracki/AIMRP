# AIMRP Architecture Overview

## 1. High-Level Goal

AIMRP enables distributed AI reasoning across peers without a central model server.
The architecture separates orchestration, peer execution, discovery, and protocol contracts.

## 2. Core Domains

### 2.1 Orchestrator Domain

Responsibilities:
- Session lifecycle (`SessionManager`)
- Task routing by role/capability (`PeerDiscoveryClient`)
- Result aggregation and consensus (`ConsensusEngine`)
- Reputation persistence (`ReputationStore`)

Input:
- User problem
- DHT-discovered peer manifests

Output:
- Final session answer
- Updated reputation signals

### 2.2 Peer Domain

Responsibilities:
- Serve protocol endpoints (`ApiServer`)
- Execute inference/planning/scoring based on role
- Publish capabilities and liveness (`CapabilitiesProvider`, `DhtPublisher`)
- Isolate model backend details (`IModelAdapter`)

Model Adapter Strategy (confirmed):
- `IModelAdapter` (domain interface)
- `OpenAiCompatibleAdapter` for Ollama, LM Studio, vLLM, OpenRouter, OpenAI, Together, Azure OpenAI
- `AnthropicAdapter` for Claude

Backend switch must be config-only (`endpoint`, `model`, `api_key`).

### 2.3 Discovery Domain (DHT)

Responsibilities:
- Store signed peer manifest records
- Resolve peers by role and capability hints
- Handle join/leave and record expiration

Design decision:
- Keep discovery behind `IDhtClient` abstraction
- Implement concrete pure .NET Kademlia in Sprint 3

## 3. Transport and Protocol

Current binding:
- HTTP/JSON-first (v0.1)

Planned evolution:
- Optional gRPC binding in later versions

Protocol message families:
- Capability exchange
- Plan generation
- Inference execution
- Scoring and critique
- Session coordination events

Critical payloads require signature validation.

## 4. Data and Trust

### 4.1 Identity

- Peer identity is public-key based (Ed25519)
- `peer_id` is derived from public key hash

### 4.2 Reputation

- Numeric score in range [-1, 1]
- Updated after scoring and consensus
- Used as weighting signal in peer selection

### 4.3 Security Baseline

- Signature checks for manifest and task outputs
- Anti-replay fields (timestamp + nonce)
- Optional deployment allowlist for bootstrap hardening

## 5. Consensus Strategy

Execution now:
- Weighted majority + reputation (v0.1 path)

Strategic target:
- BFT-capable consensus module (v0.2)

Architectural implication:
- `ConsensusEngine` must be interface-driven so algorithms can be swapped without changing session API contracts.

## 6. Runtime Topology (Conceptual)

1. Peer nodes publish signed manifests into DHT.
2. Orchestrator discovers peers by role and latency/model constraints.
3. Orchestrator starts a session and obtains a plan.
4. Tasks are executed by reasoner/retriever peers.
5. Critics score candidate answers.
6. Consensus produces final output.
7. Reputation updates persist for future routing.

## 7. Non-Goals in Sprint 1

- Production network hardening
- Final BFT wire protocol
- Billing, quota, and payment layer
- Performance benchmark commitments

## 8. Sprint Hand-off to Sprint 2

Sprint 2 should formalize:
- JSON schemas or Protobuf contracts for all payloads
- Error model and status codes
- Envelope metadata for signatures, timestamps, and protocol version
