# Changelog

All notable changes to AIMRP are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-05-03

First standardized release of AIMRP. Closes the RC1 review and freezes the
0.1 wire format for interoperable peers and orchestrators.

### Added

- **Protocol versioning** propagated end-to-end:
  - `protocol_version` field added to `InferRequest`, `InferResponse`,
    `PlanRequest`, `PlanResponse`, `ScoreRequest`, `ScoreResponse`,
    `CapabilitiesRequest`, and `PeerManifest` (covered by signature).
  - Mirrors the `AIMRP-Version` HTTP header per RFC §6.0.
- **`Error` message** in `proto/aimrp.proto`: stable, transport-agnostic
  failure envelope embedded in `InferResponse`, `PlanResponse`,
  `ScoreResponse`. Existing `ErrorResponse` remains the HTTP envelope.
- **Optional sampling fields** on `/infer` (`temperature`, `top_p`, `stop`,
  `model_name`) and corresponding section "Model Selection Rules" in
  `docs/api-peer.md`.
- **Orchestrator retry policy** (exponential backoff with full jitter) and
  full `PeerFilter` definition in `docs/api-orchestrator.md`.
- **ConsensusEngine input JSON example** in `docs/api-orchestrator.md`.
- **DHT additions** in `docs/dht-design.md`:
  - Canonical JSON manifest example.
  - `DhtLookupRequest` / `DhtLookupResponse` JSON wire shapes.
  - 8 KiB record-size budget table.
  - Peer Manifest Lifecycle ASCII diagram.
  - Retry/backoff section for join and refresh.
- **Consensus design additions** in `docs/consensus-design.md`:
  - Weighted majority and PBFT flow diagrams.
  - Worked numerical example.
  - Normative `low_confidence_threshold` (default `0.35`).
  - "Reputation in BFT mode" section.
- **Peer lifecycle additions** in `docs/peer-lifecycle.md`:
  - Lifecycle state diagram (`INIT → STARTING → RUNNING ⇄ DEGRADED →
    DRAINING → STOPPED`).
  - Model healthcheck section.
  - Dynamic role change rules.
  - Retry/backoff for DHT join and refresh.
- **Peer implementation guide additions** in
  `docs/peer-implementation-guide.md`:
  - Minimal Peer Architecture diagram.
  - Reference `OpenAiCompatibleAdapter : IModelAdapter` implementation.
  - "Common pitfalls" table.
- **New example configs** in `examples/sample-configs/`:
  - `peer.planner.yaml`, `peer.reasoner.yaml`, `peer.critic.yaml`,
    `peer.retriever.yaml`.
  - `orchestrator.full.yaml` (full configurable surface).
  - `docker-compose.local-cluster.yaml` (planner + 2× reasoner + critic +
    orchestrator + bootstrap + Ollama).
  - `example-session.json` (end-to-end session record).

### Changed

- Bumped all document headers from `Version: 0.1` to `Version: 0.1.0`
  across `docs/api-peer.md`, `docs/api-orchestrator.md`,
  `docs/consensus-design.md`, `docs/dht-design.md`,
  `docs/peer-lifecycle.md`, `docs/peer-implementation-guide.md`,
  `docs/state-machine.md`, `docs/reference-implementation.md`,
  `docs/operational-guidelines.md`.
- `PeerManifest.protocol_version` is now signature-covered, preventing
  DHT-layer protocol-version downgrade attacks.

### Conformance

- Conformance levels (`aimrp-open`, `aimrp-safe`, `aimrp-strict`) remain as
  defined in RFC §21.
- Implementations claiming `aimrp-safe` or `aimrp-strict` MUST honor the
  `protocol_version` field on every signed payload and MUST reject mismatched
  versions with `version_unsupported`.

### Notes

- The deeper RFC sections (Identity & Security §3, DHT §4, Session Protocol
  §7, Consensus §9, Security §11, Error Codes §14, Compliance §21,
  Appendices A–G) were already standardization-complete prior to 0.1.0 and
  were not modified in this release.
