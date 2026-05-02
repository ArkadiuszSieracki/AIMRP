# AIMRP — AI Mesh Reasoning Protocol

**Conceptual design / RFC stage. Not yet implemented.**

AIMRP is an open protocol specification for distributed AI reasoning across a peer-to-peer network. It defines how independent AI nodes discover each other, negotiate roles, exchange signed messages, and produce consensus answers — without a central server.

## What problem does it solve?

Running large reasoning tasks on a single AI node is a bottleneck: one model, one context window, one point of failure. AIMRP distributes the reasoning across multiple peers, each contributing a specialized role — planner, reasoner, critic, or retriever — and aggregating results through a consensus layer.

## How the network works

```
┌──────────────────────────────────────────────────────┐
│                   AIMRP Network                       │
│                                                       │
│  ┌─────────┐   SESSION_INIT    ┌─────────────────┐   │
│  │  Client │ ────────────────► │  Orchestrator   │   │
│  └─────────┘                   │  (coordinator)  │   │
│                                └────────┬────────┘   │
│                                         │             │
│                 ┌───────────────────────┤             │
│                 │                       │             │
│          ┌──────▼──────┐        ┌───────▼──────┐     │
│          │  Peer A     │        │  Peer B       │     │
│          │  (planner   │        │  (reasoner +  │     │
│          │  +reasoner) │        │   critic)     │     │
│          └─────────────┘        └───────────────┘     │
│                                                       │
│              Kademlia DHT (peer discovery)            │
└──────────────────────────────────────────────────────┘
```

1. Peers join the network by publishing a signed manifest to the DHT.
2. The orchestrator looks up peers by role, checks liveness, and opens a session.
3. The planner decomposes the goal into task steps.
4. Reasoners and retrievers execute tasks in parallel.
5. Critics score each answer against defined criteria.
6. The orchestrator runs weighted-majority consensus and returns the final result.
7. Reputation scores are updated based on critic feedback.

## Protocol highlights

| Feature | Detail |
|---|---|
| Transport | HTTP/JSON (v0.1); gRPC defined in proto for future use |
| Identity | Ed25519 keypairs; `peer_id = SHA-256(pubkey)` |
| Discovery | Kademlia DHT with per-role secondary indexes |
| Roles | planner, reasoner, critic, retriever |
| Consensus | Weighted majority (v0.1); PBFT roadmap (v0.2) |
| Versioning | `X-AIMRP-Version` header; version negotiation built-in |
| Security | Signed messages, anti-replay (timestamp + nonce), threat model in RFC |
| Model backends | Any OpenAI-compatible API (Ollama, vLLM, OpenAI, Azure, …) or Anthropic Claude |

## Repository structure

```
docs/
  rfc/RFC-AIMRP-0.1.md          — primary protocol specification
  api-peer.md                   — HTTP API reference: peer node
  api-orchestrator.md           — HTTP API reference: orchestrator
  dht-design.md                 — DHT design (Kademlia, IDhtClient)
  consensus-design.md           — consensus algorithm design
  peer-lifecycle.md             — peer startup/shutdown/error lifecycle
  peer-implementation-guide.md  — step-by-step: build your own peer

proto/
  aimrp.proto                   — canonical message schema + gRPC service definitions

orchestrator/
  README.md                     — orchestrator module design

peer/
  README.md                     — peer node module design

tools/cli/
  README.md                     — CLI spec (aimrp commands)

examples/sample-configs/
  peer.local.yaml               — local dev peer config (Ollama)
  peer.cloud.yaml               — cloud peer config (OpenAI GPT-4o)
  orchestrator.local.yaml       — local orchestrator config
```

## Sprint roadmap

| Sprint | Status | Deliverables |
|---|---|---|
| Bootstrap | ✅ Done | `.mastermind/` layer, architectural decisions |
| 1 | ✅ Done | RFC v0.1, architecture overview, session flow examples |
| 2 | ✅ Done | `aimrp.proto`, peer API, orchestrator API |
| 3 | ✅ Done | DHT design, `IDhtClient`, DhtService |
| 4 | ✅ Done | Orchestrator design, consensus design |
| 5 | ✅ Done | Peer node design, lifecycle, sample configs |
| 6 | ✅ Done | CLI spec, project README, peer implementation guide |
| 7+ | ⏳ Pending | .NET implementation, tests, integration |

## License

Specification only. Implementation license TBD.
