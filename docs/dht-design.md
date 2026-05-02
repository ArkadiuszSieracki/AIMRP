# AIMRP DHT Design

Version: 0.1  
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
