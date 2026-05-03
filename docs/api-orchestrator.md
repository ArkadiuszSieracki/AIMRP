# AIMRP Orchestrator Internal Design Reference

Version: 0.1.0

This document describes the orchestrator's internal flow and the contracts it relies on
when interacting with peers. It is not a public HTTP API but a design reference for Sprint 4 implementation.

## 1. Session Lifecycle

```
CREATE SESSION
     │
     ▼
DISCOVER PEERS (by role via DHT)
     │
     ▼
SESSION_INIT: call planner /plan
     │
     ▼
for each step:
  TASK_ASSIGN: call reasoner/retriever /infer
     │
     ▼
  TASK_EVAL: call critic /score for each answer
     │
     ▼
CONSENSUS: aggregate scores + reputation weights
     │
     ▼
EMIT FINAL ANSWER + UPDATE REPUTATION
```

## 2. Peer Selection Policy

Orchestrator MUST select peers using these rules (in order):

1. Role match — peer must advertise the required role.
2. Reputation threshold — peer reputation score MUST be > configured minimum (default: -0.5).
3. Latency hint — prefer peers with lower `avg_latency_ms` if multiple candidates qualify.
4. Availability — peer must have responded to liveness check within window.

If no peer qualifies for a required role, orchestrator MUST return an error for that session task.

## 3. ConsensusEngine Contract

Input per task:
- List of `(peer_id, completion, confidence)` from reasoners
- List of `(peer_id, scores, overall)` from critics
- Current reputation scores per peer

**Input JSON example:**

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "candidates": [
    {
      "peer_id": "a3f9c2...",
      "completion": "RAFT is a leader-based consensus algorithm...",
      "confidence": 0.87,
      "reputation_score": 0.42,
      "critic_scores": [
        { "critic_peer_id": "b1c4d6...", "overall": 0.88, "per_criterion": { "logic": 0.9, "factuality": 0.86 } },
        { "critic_peer_id": "c8e2a1...", "overall": 0.82, "per_criterion": { "logic": 0.85, "factuality": 0.79 } }
      ]
    },
    {
      "peer_id": "d2b7f9...",
      "completion": "RAFT uses a single elected leader...",
      "confidence": 0.74,
      "reputation_score": 0.10,
      "critic_scores": [
        { "critic_peer_id": "b1c4d6...", "overall": 0.71, "per_criterion": { "logic": 0.75, "factuality": 0.68 } }
      ]
    }
  ]
}
```

Output:
- Winning completion (string)
- Consensus score (float)
- Confidence-weighted rationale (optional)

v0.1 algorithm — weighted majority:

```
for each candidate answer:
  weight = avg(critic_overall) × reputation_score × confidence
final = candidate with max(weight)
```

v0.2 target — BFT-capable interface:
- `ConsensusEngine` MUST be injectable/swappable (interface-driven).
- BFT algorithm implementation replaces only the `ConsensusEngine` component.

## 4. Reputation Update Contract

After each session task evaluation:

```
delta = f(critic_score, majority_agreement)
new_score = clamp(old_score + delta, -1.0, 1.0)
```

Update is persisted to `ReputationStore` by `peer_id`.

Suggested delta function (v0.1):
- `delta = (critic_overall - 0.5) × 0.1` — small incremental updates

## 5. Error Handling Policy

| Situation | Action |
|---|---|
| Peer timeout on `/infer` | Select backup peer from DHT; re-issue task with same `task_id` |
| Signature invalid on TASK_RESULT | Reject response; apply negative reputation delta |
| No peers available for role | Fail session with `peer_unavailable` |
| ConsensusEngine returns no winner | Fail task; log with `internal_error` |
| Session not found | Return `session_not_found` to caller |

### 5.1 Retry Policy (Normative)

All outbound peer calls (`/plan`, `/infer`, `/score`, `/capabilities`) MUST follow exponential backoff with full jitter:

```
base_delay_ms     = 200
max_delay_ms      = 5000
max_retries       = 3
attempt_delay(n)  = random_uniform(0, min(max_delay_ms, base_delay_ms * 2^n))
total_call_budget = 15_000 ms
```

Rules:

- Retries apply only to: HTTP 5xx, network errors, `peer_unavailable`, `rate_limited`.
- `rate_limited` responses MUST honor `Retry-After` header (seconds) over the computed backoff if larger.
- `invalid_request`, `signature_invalid`, `version_unsupported`, `unsafe_prompt`, `model_error` are NOT retried; the orchestrator MUST select a different peer.
- Same `task_id` MUST be reused on retry against the same peer; a new `task_id` MUST be issued only when failing over to a different peer.
- The total wall-clock budget for a single task (across all retries and failovers) is `total_call_budget`. Exceeding it returns `peer_unavailable` to the session.

## 6. Interfaces for Sprint 4 Implementation

```csharp
// Peer discovery abstraction — wraps IDhtClient with orchestrator-level policy
interface IPeerDiscoveryClient
{
    // Finds peers by role and optional filter via IDhtClient lookup.
    // Applies reputation threshold and liveness check before returning results.
    Task<IReadOnlyList<PeerManifest>> FindPeersAsync(Role role, PeerFilter? filter = null);
}

// DHT abstraction — defined in dht-design.md; consumed by IPeerDiscoveryClient
interface IDhtClient
{
    Task PublishAsync(PeerManifest manifest, CancellationToken ct = default);
    Task<IReadOnlyList<PeerManifest>> LookupAsync(PeerFilter filter, int maxResults = 10, CancellationToken ct = default);
    Task JoinAsync(string bootstrapAddress, CancellationToken ct = default);
    Task LeaveAsync(CancellationToken ct = default);
}

// Consensus abstraction (supports v0.1 weighted and future BFT)
interface IConsensusEngine
{
    Task<ConsensusResult> ResolveAsync(IReadOnlyList<CandidateAnswer> candidates);
}

// Reputation abstraction
interface IReputationStore
{
    Task<double> GetScoreAsync(string peerId);
    Task UpdateAsync(string peerId, double delta);
}

// Session manager abstraction
interface ISessionManager
{
    Task<Session> CreateAsync(string problem);
    Task<Session> GetAsync(string sessionId);
    Task CompleteAsync(string sessionId, string finalAnswer);
}
```

## 7. Hand-off to Sprint 4

Sprint 4 must produce:
- Concrete implementation of all interfaces above
- `orchestrator/README.md` with module diagram
- `docs/consensus-design.md` with PBFT/HotStuff comparison and decision
- Sequence diagrams for SESSION_INIT → CONSENSUS flow

## 8. PeerFilter Definition

`PeerFilter` is the structured query passed to `IDhtClient.LookupAsync` and `IPeerDiscoveryClient.FindPeersAsync`.

```csharp
public sealed record PeerFilter
{
    public required Role   Role            { get; init; } // REQUIRED; exact match
    public uint            MaxLatencyMs    { get; init; } = 0;     // 0 = no constraint
    public string          ModelNameHint   { get; init; } = "";    // substring match; "" = no constraint
    public double          MinReputation   { get; init; } = -1.0;  // [-1.0, 1.0]
    public string?         ComplianceLevel { get; init; }          // null = any; otherwise exact match
}
```

Filter semantics MUST match those documented in `docs/dht-design.md` §5.2. Unknown fields in serialized form (forward compatibility) MUST be ignored by the consumer.

**JSON wire shape:**

```json
{
  "role": "reasoner",
  "max_latency_ms": 300,
  "model_name_hint": "llama",
  "min_reputation": 0.0,
  "compliance_level": "aimrp-safe"
}
```
