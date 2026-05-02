# AIMRP Orchestrator Internal Design Reference

Version: 0.1

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
