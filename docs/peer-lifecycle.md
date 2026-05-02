# AIMRP Peer Lifecycle

Version: 0.1  
Sprint: 5

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
4. Join DHT network
   - DhtClient.JoinAsync(bootstrap_address)
5. Publish manifest
   - DhtClient.PublishAsync(signed manifest)
6. Start HTTP API server
   - listen on config.network.listen_address
7. Start DHT refresh loop
   - publish updated manifest every (ttl_seconds / 2)
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
