# AIMRP DHT Design

Version: 0.1.0  
Sprint: 3

## 1. Purpose

The DHT layer provides peer discovery without a central registry.
Peers publish signed manifests; orchestrators query by role and capability constraints.

## 2. Algorithm: Kademlia (Pure .NET)

AIMRP uses a Kademlia-compatible DHT design.

Implementation decision:
- Pure .NET Kademlia library behind `IDhtClient` abstraction.
- Concrete implementation deferred to Sprint 3 implementation phase.
- Interface contract defined in proto (`DhtService`) and C# (`IDhtClient`).

Kademlia key properties used:
- XOR metric for peer routing
- k-buckets for peer routing table
- Iterative node lookup for peer discovery

## 3. IDhtClient Interface

```csharp
public interface IDhtClient
{
    // Publish or refresh this peer's signed manifest in the DHT.
    Task PublishAsync(PeerManifest manifest, CancellationToken ct = default);

    // Find peers matching the given filter. Returns up to maxResults manifests.
    Task<IReadOnlyList<PeerManifest>> LookupAsync(PeerFilter filter, int maxResults = 10, CancellationToken ct = default);

    // Connect to the DHT network via a known bootstrap node.
    Task JoinAsync(string bootstrapAddress, CancellationToken ct = default);

    // Gracefully leave the DHT (withdraw manifest).
    Task LeaveAsync(CancellationToken ct = default);
}
```

## 4. Peer Manifest Record

### 4.1 DHT Key

Key format: `aimrp:peer:{peer_id}`

`peer_id` is the hex-encoded SHA-256 of the peer's Ed25519 public key.

### 4.2 Record Fields

| Field | Type | Description |
|---|---|---|
| `peer_id` | string | Hash of public key |
| `pubkey` | bytes | Ed25519 public key |
| `endpoints` | Endpoint[] | `{ type, address }` per endpoint |
| `roles` | Role[] | Roles this peer can perform |
| `models` | ModelInfo[] | Models available via this peer |
| `reputation_score` | double | [-1.0, 1.0] |
| `reputation_samples` | uint32 | Sample count for score confidence |
| `timestamp` | int64 | Unix seconds of last publish |
| `nonce` | string | Anti-replay random value |
| `ttl_seconds` | uint32 | Record lifetime; 0 = network default (3600s) |
| `signature` | bytes | Ed25519 signature of all other fields |

### 4.3 Record Lifecycle

- **Publish:** Peer signs manifest and writes to DHT on startup.
- **Refresh:** Peer republishes before TTL expires (recommended: TTL/2).
- **Expiry:** Records past TTL are considered stale and excluded from lookup results.
- **Withdraw:** Peer removes record on graceful shutdown (if DHT supports explicit delete).

Default TTL: 3600 seconds. Minimum: 60 seconds.

### 4.5 Example Manifest (Canonical JSON)

```json
{
  "endpoints": [{ "address": "peer-eu-1.example.org:8080", "type": "http-json" }],
  "models": [
    {
      "avg_latency_ms": 180,
      "max_context": 8192,
      "max_tokens": 1024,
      "name": "llama-3.2-3b"
    }
  ],
  "nonce": "f6a4b9e0-1c2d-4e5b-9a3f-7c8d1e2b0a4c",
  "peer_id": "a3f9c2d1e5b8f4a7c0d3e6b9f2a5c8d1e4b7f0a3c6d9e2b5f8a1c4d7e0b3f6a9",
  "protocol_version": "0.1",
  "pubkey": "oXk2mIQ2k0Qo8u8vPGrqkqL3fXxKqzWqGv1d2yFq0YA=",
  "reputation_samples": 17,
  "reputation_score": 0.42,
  "roles": ["planner", "reasoner"],
  "timestamp": "2026-05-02T10:15:30Z",
  "ttl_seconds": 3600,
  "signature": "MEUCIQDx...=="
}
```

Keys are sorted alphabetically per the canonical serialization algorithm (RFC §3.2.1). The `signature` field is excluded from the bytes that are signed but is present in the published record.

### 4.6 Record Size Budget

A conformant manifest MUST fit within Kademlia STORE limits.

| Component | Typical bytes | Cap |
|---|---|---|
| Header fields (peer_id, pubkey, timestamp, nonce, signature) | ~280 | — |
| `endpoints[]` (1 entry, http-json) | ~70 | 4 entries |
| `roles[]` (1–4 entries) | ~40 | — |
| `models[]` (1 entry) | ~150 | 8 entries |
| Reputation block | ~40 | — |

Maximum serialized size of a `PeerManifest` MUST NOT exceed **8 KiB** (8192 bytes) in canonical JSON. DHT nodes MUST reject larger STORE requests with `manifest_invalid`. Serialized proto3 binary form is RECOMMENDED for nodes near the cap.

### 4.7 Manifest Lifecycle Diagram

```
             ┌────────────────────────┐
             │  generate / load   │
             │   Ed25519 keypair  │
             └─────────┬──────────┘
                       ▼
             ┌────────────────────────┐
             │  build PeerManifest│
             │  + protocol_version│
             │  + nonce + ts      │
             └─────────┬──────────┘
                       ▼
             ┌────────────────────────┐
             │ sign(canonical(M)) │
             └─────────┬──────────┘
                       ▼
             ┌────────────────────────┐
             │   PUBLISHED        │◀──────────┐
             │   (in DHT)         │           │
             └─────────┬──────────┘           │
               every TTL/2                       │ refresh
                       │ (re-sign with new      │ (timestamp
                       │  timestamp + nonce)    │  + nonce)
                       └──────────────────────────────┘
                       │
     graceful shutdown / TTL expiry
                       ▼
             ┌────────────────────────┐
             │ EXPIRED / WITHDRAWN │
             └────────────────────────┘
```

### 4.4 Signature Coverage

Signature MUST cover the concatenation (canonical JSON or proto binary) of all fields except `signature` itself.
Consumers MUST verify signature before using any manifest data.

## 5. Lookup Semantics

### 5.1 Logical Query Examples

```
find_peers(role=PLANNER)
find_peers(role=REASONER, max_latency_ms<300)
find_peers(role=CRITIC, model_name_hint~"llama", min_reputation>0.2)
```

**Wire shape (DhtLookupRequest, JSON):**

```json
{
  "filter": {
    "role": "reasoner",
    "max_latency_ms": 300,
    "model_name_hint": "llama",
    "min_reputation": 0.2
  },
  "max_results": 10
}
```

**Example response:**

```json
{
  "peers": [
    { "peer_id": "a3f9c2...", "endpoints": [{ "type": "http-json", "address": "peer-eu-1.example.org:8080" }], "roles": ["reasoner"], "reputation_score": 0.42 }
  ]
}
```

### 5.2 PeerFilter Fields

| Field | Effect |
|---|---|
| `role` | Required. Only peers advertising this role are returned. |
| `max_latency_ms` | 0 = no constraint. Filters by manifest `avg_latency_ms`. |
| `model_name_hint` | "" = no constraint. Substring match on model name. |
| `min_reputation` | Default -1.0 (no filter). Excludes peers below threshold. |

### 5.3 Implementation Mapping

Kademlia does not natively support attribute queries. AIMRP uses secondary indexes:

- Per-role bucket: `aimrp:role:{role_name}` → set of peer_ids
- Orchestrator performs lookup: get peer_ids from role bucket, then fetch each manifest, apply remaining filters client-side.

## 6. Join / Leave Flow

### 6.1 Join

```
GENERATE keypair (if not already present)
  │
  ▼
CALL DhtClient.JoinAsync(bootstrapAddress)
  │
  ▼
BOOTSTRAP: exchange k-bucket entries with bootstrap node
  │
  ▼
SIGN manifest
  │
  ▼
CALL DhtClient.PublishAsync(manifest)
  │
  ▼
PEER IS DISCOVERABLE
```

### 6.2 Refresh Loop

```
every (TTL / 2) seconds:
  refresh manifest timestamp and nonce
  re-sign
  re-publish
```

### 6.2.1 Retry / Backoff for Join and Refresh

Join and refresh failures MUST follow exponential backoff with jitter, identical to the orchestrator retry policy (api-orchestrator.md §5.1):

```
base_delay_ms = 1000
max_delay_ms  = 60_000
max_attempts  = unlimited (refresh)
max_attempts  = 5         (initial join)
attempt_delay(n) = random_uniform(0, min(max_delay_ms, base_delay_ms * 2^n))
```

If initial join exceeds `max_attempts`, peer MUST fail startup with `bootstrap_failed` (RFC §4.4). If refresh fails for the entire TTL window, the manifest MUST be considered expired by other peers; the local peer MUST keep retrying with backoff until success or shutdown.

### 6.3 Leave

```
CALL DhtClient.LeaveAsync()
  │
  ▼
WITHDRAW manifest record (if DHT supports delete)
  │
  ▼
CLOSE network connections
```

## 7. Sequence: Orchestrator Lookup

```
Orchestrator                  DhtClient                   DHT Network
    │                             │                             │
    │ LookupAsync(filter)         │                             │
    │────────────────────────────>│                             │
    │                             │ get_peer_ids(role bucket)   │
    │                             │────────────────────────────>│
    │                             │ [peer_id_1, peer_id_2, ...] │
    │                             │<────────────────────────────│
    │                             │ fetch_manifest(peer_id_1)   │
    │                             │────────────────────────────>│
    │                             │ PeerManifest                │
    │                             │<────────────────────────────│
    │                             │ verify_signature(manifest)  │
    │                             │─────────┐                   │
    │                             │<────────┘                   │
    │                             │ apply PeerFilter            │
    │                             │─────────┐                   │
    │                             │<────────┘                   │
    │ [PeerManifest, ...]         │                             │
    │<────────────────────────────│                             │
```

## 8. Security Notes

- All manifests MUST be signature-verified before use.
- Orchestrators MUST ignore manifests with expired timestamps (now - timestamp > TTL).
- Nonce value MUST differ on each publish to prevent replay.
- Sybil mitigation: reputation warm-up period + bootstrap allow-list policy per deployment.

## 9. Open Items

- Concrete Kademlia .NET library selection (research: Kademlia.Net, custom implementation)
- k-bucket size parameter (k=20 is Kademlia standard)
- Alpha concurrency parameter for parallel lookups
- Bootstrap node configuration format
- DHT explicit delete support (graceful leave)

## 10. Hand-off to Sprint 4

Sprint 4 (orchestrator design) will consume:
- `IDhtClient.LookupAsync` for peer selection in `PeerDiscoveryClient`
- `PeerManifest` as the peer descriptor in `IPeerDiscoveryClient` return type
- `PeerFilter` as the query input from `SessionManager`
