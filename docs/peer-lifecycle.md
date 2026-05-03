# AIMRP Peer Lifecycle

Version: 0.1.0  
Sprint: 5

## 0. Lifecycle Overview

```
   ┌─────────┐   load_config + keypair    ┌────────────┐
   │  INIT   │ ─────────────────────▶ │  STARTING  │
   └─────────┘                          └─────│──────┘
                                            healthcheck model
                                            DHT join + publish
                                                  ▼
                                            ┌─────────────┐
   ┌─────────────┐   model down       │  RUNNING    │◄────┐
   │ DEGRADED    │ ◀───────────────── │  (steady)   │    │
   │ (4xx infer) │   model recovers │             │    │ refresh
   └─────│──────┘ ─────────────────▶ └─────┬───────┘    │ (TTL/2)
         │                                  │           ────┘
         │ SIGTERM                          │ SIGTERM
         ▼                                  ▼
   ┌─────────────┐ drain in-flight     ┌───────────────┐
   │  DRAINING   │ ───────────────────▶ │  STOPPED      │
   └─────────────┘ DHT leave           └───────────────┘
```

State set: `INIT → STARTING → RUNNING ⇄ DEGRADED → DRAINING → STOPPED`. Implementations MUST expose the current state via internal logging and SHOULD expose it via a local `/healthz` endpoint (out of normative scope).

## 1. Startup Sequence

```
1. Load config (peer.yaml / env vars)
2. Load or generate Ed25519 keypair
   - key_path from config
   - if file missing: generate new keypair, write to key_path
   - peer_id = hex(SHA-256(pubkey))
3. Build PeerManifest
   - roles, models, endpoints from config
   - timestamp = UTC now (Unix seconds)
   - nonce = random UUID
   - ttl_seconds from config (default 3600)
   - sign manifest with private key
4. Healthcheck model backend
   - call IModelAdapter.IsAvailableAsync()
   - on failure: log warning; transition to DEGRADED
   - on success: transition continues
5. Join DHT network
   - DhtClient.JoinAsync(bootstrap_address)
   - retry with exponential backoff (base 1s, max 60s, max_attempts=5)
   - on full failure: exit with `bootstrap_failed`
6. Publish manifest
   - DhtClient.PublishAsync(signed manifest)
7. Start HTTP API server
   - listen on config.network.listen_address
8. Start DHT refresh loop
   - publish updated manifest every (ttl_seconds / 2)
   - on refresh failure: exponential backoff (base 1s, max 60s, unbounded retries)
```

## 2. Steady State

```
While running:
  ├── Serve API requests (GET /capabilities, POST /plan, POST /infer, POST /score)
  ├── Execute model calls via IModelAdapter on each task
  ├── Sign all critical responses (InferResponse, PlanResponse, ScoreResponse)
  └── Refresh DHT manifest every (TTL/2) seconds
```

## 3. Shutdown Sequence

```
1. Receive shutdown signal (SIGTERM / Ctrl+C / cancellation token)
2. Stop accepting new API requests
3. Drain in-flight requests (with timeout)
4. Leave DHT: DhtClient.LeaveAsync()
5. Dispose model adapter and HTTP server
6. Exit cleanly (code 0)
```

## 4. Error Scenarios

| Scenario | Behavior |
|---|---|
| Model backend unavailable on startup | Log error; peer starts but returns `peer_unavailable` on /infer until backend recovers |
| DHT bootstrap unreachable | Log warning; retry join with backoff; peer is not discoverable until DHT joined |
| DHT refresh fails | Log warning; retry next interval; manifest expires if refresh fails for full TTL |
| Keypair file missing | Generate new keypair; log warning that peer_id will change |
| Inbound request with wrong API version | Return HTTP 400 with error code `version_unsupported` |
| Signature verification fails on inbound (future) | Reject request; return `signature_invalid` |

## 5. Role Eligibility Rules

| Role | Requirement |
|---|---|
| PLANNER | Peer must advertise role and model must support structured output (steps list) |
| REASONER | Peer must advertise role; any model capable of instruction following |
| CRITIC | Peer must advertise role; model must support per-criterion scoring output |
| RETRIEVER | Peer must advertise role; model used for embedding or retrieval synthesis |

A peer advertising a role is responsible for returning valid role-appropriate responses.
If a peer cannot fulfill a role task (e.g., model error), it returns `model_error`.

## 6. Identity Continuity

- A peer's identity is its `peer_id` (SHA-256 of public key).
- Changing the keypair file changes the identity.
- Reputation scores in the orchestrator are keyed by `peer_id` — a new keypair starts with reputation 0.
- Keep keypair files backed up and persistent across restarts.

## 7. Liveness Model

- Peers are considered live while their DHT manifest is within TTL.
- Orchestrators may perform liveness checks (GET /capabilities before task assignment).
- If a peer fails to respond within `liveness_window_seconds`, it is excluded from routing for that session.

## 8. Model Healthcheck

Peers MUST periodically validate the model backend:

- **Interval:** every `health_interval_seconds` (default 30s).
- **Probe:** `IModelAdapter.IsAvailableAsync()` — lightweight call (e.g. list-models, /healthz, or a 1-token completion).
- **Failure handling:**
  - 1–2 consecutive failures → log warning, stay RUNNING.
  - ≥ 3 consecutive failures → transition to DEGRADED, return `peer_unavailable` on `/infer`, `/plan`, `/score`.
  - On recovery (1 successful probe) → transition back to RUNNING.
- **Manifest impact:** while in DEGRADED, the peer SHOULD continue refreshing its manifest (so it remains discoverable for capability queries) but `/capabilities` MUST report `compliance_level` unchanged. AIMRP-Strict deployments MAY withdraw the manifest while degraded.

## 9. Dynamic Role Change

A peer MAY change its advertised roles at runtime (e.g. operator adds `critic` to a previously reasoner-only peer):

1. Update local config (`roles:` list).
2. Rebuild `PeerManifest` with new `roles[]`, fresh `nonce`, current `timestamp`.
3. Re-sign and publish via `DhtClient.PublishAsync`.
4. Begin accepting requests for the new role at the next inbound request after publication.

Normative constraints:

- Role changes MUST NOT change `peer_id` (same keypair).
- A peer MUST NOT remove a role while it has in-flight tasks for that role; drain first.
- After a role addition, the orchestrator may not discover the new capability until it re-queries the DHT (Kademlia eventual consistency).
- Role removal does not revoke historical reputation tied to that role.
