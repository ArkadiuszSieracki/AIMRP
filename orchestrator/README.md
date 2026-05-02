# AIMRP Orchestrator — Design Document

Version: 0.1  
Sprint: 4

## 1. Overview

The orchestrator is the session coordinator in the AIMRP network. It is a standalone process that:
- Discovers peers via DHT
- Decomposes problems into tasks using planner peers
- Assigns tasks to reasoner and retriever peers
- Evaluates outputs using critic peers
- Runs consensus to produce final answers
- Updates peer reputation scores

The orchestrator does NOT serve a public HTTP API. It is invoked by a client or CLI. All communication with peers uses the Peer API defined in docs/api-peer.md.

## 2. Module Diagram

```
OrchestratorApp
  │
  ├── SessionManager           — creates, tracks, and completes sessions
  │     └── uses: IPeerDiscoveryClient, IConsensusEngine, IReputationStore
  │
  ├── PeerDiscoveryClient      — wraps IDhtClient with policy layer
  │     └── uses: IDhtClient, reputation threshold, liveness check
  │
  ├── ConsensusEngine          — resolves candidate answers to final output
  │     └── v0.1: weighted majority; v0.2 interface: IConsensusEngine (BFT-ready)
  │
  └── ReputationStore          — persists and updates peer reputation scores
```

## 3. Session Lifecycle

```
Client Request
     │
     ▼
SessionManager.CreateAsync(problem)
     │  → generates session_id (UUID)
     ▼
PeerDiscoveryClient.FindPeersAsync(PLANNER)
     │  → fetches signed manifests from DHT
     │  → applies reputation threshold and liveness filter
     ▼
PeerClient.Plan(plannerPeer, session_id, problem)
     │  → POST /plan to selected planner peer
     │  → verifies signature on PlanResponse
     ▼
foreach step in plan.Steps:
  │
  ├── PeerDiscoveryClient.FindPeersAsync(step.SuggestedRole)
  │
  ├── PeerClient.Infer(peer, session_id, task_id, prompt)
  │     → POST /infer; verifies signature on InferResponse
  │
  └── PeerClient.Score(criticPeer, session_id, task_id, answer, criteria)
        → POST /score; verifies signature on ScoreResponse
     │
     ▼
ConsensusEngine.ResolveAsync(candidates)
     │  → selects final answer
     ▼
ReputationStore.UpdateAsync(peer_id, delta) — for each evaluated peer
     │
     ▼
SessionManager.CompleteAsync(session_id, finalAnswer)
```

## 4. Peer Discovery Policy

`PeerDiscoveryClient` applies the following rules before returning peers:

| Order | Rule | Default |
|---|---|---|
| 1 | Role match | Required — no override |
| 2 | Reputation threshold | min_reputation ≥ -0.5 |
| 3 | Latency hint | prefer lower avg_latency_ms |
| 4 | Liveness | must have responded within liveness window (configurable) |

If no peer qualifies, the orchestrator returns an error for that session task.

## 5. ConsensusEngine — v0.1 Weighted Majority

### Input

For each task:
- `List<CandidateAnswer>`: `(peer_id, completion, confidence)`
- `List<CriticScore>`: `(peer_id, overall)`
- `Dictionary<string, double>` reputation per peer_id

### Algorithm

```
for each candidate in candidates:
    critic_score = avg(scores where peer scored this candidate)
    reputation   = reputationStore[candidate.peer_id]  // clamped [-1, 1]
    weight       = critic_score × reputation_normalized × candidate.confidence

    // reputation_normalized = (reputation + 1) / 2  → maps [-1,1] to [0,1]

winner = candidate with max(weight)
```

### Output

```csharp
record ConsensusResult(
    string WinningCompletion,
    double ConsensusScore,      // winner's weight, normalized [0,1]
    string WinnerPeerId
);
```

### Interface (BFT-ready)

```csharp
interface IConsensusEngine
{
    Task<ConsensusResult> ResolveAsync(
        IReadOnlyList<CandidateAnswer> candidates,
        CancellationToken ct = default);
}
```

v0.2 replaces the implementation behind this interface with a BFT algorithm (PBFT or HotStuff — see docs/consensus-design.md).

## 6. ReputationStore

### Update Formula

```
delta = (critic_overall - 0.5) × 0.1
new_score = clamp(old_score + delta, -1.0, 1.0)
```

### Interface

```csharp
interface IReputationStore
{
    Task<double> GetScoreAsync(string peerId);
    Task UpdateAsync(string peerId, double delta);
}
```

### Persistence

- v0.1: in-memory with optional JSON file persistence
- Future: EF Core + SQLite (see .mastermind/skills/ef-core-db-review/SKILL.md)

## 7. C# Interfaces Summary

```csharp
interface ISessionManager
{
    Task<Session> CreateAsync(string problem, CancellationToken ct = default);
    Task<Session> GetAsync(string sessionId, CancellationToken ct = default);
    Task CompleteAsync(string sessionId, string finalAnswer, CancellationToken ct = default);
}

interface IPeerDiscoveryClient
{
    Task<IReadOnlyList<PeerManifest>> FindPeersAsync(
        Role role,
        PeerFilter? filter = null,
        CancellationToken ct = default);
}

interface IDhtClient
{
    Task PublishAsync(PeerManifest manifest, CancellationToken ct = default);
    Task<IReadOnlyList<PeerManifest>> LookupAsync(PeerFilter filter, int maxResults = 10, CancellationToken ct = default);
    Task JoinAsync(string bootstrapAddress, CancellationToken ct = default);
    Task LeaveAsync(CancellationToken ct = default);
}

interface IConsensusEngine
{
    Task<ConsensusResult> ResolveAsync(IReadOnlyList<CandidateAnswer> candidates, CancellationToken ct = default);
}

interface IReputationStore
{
    Task<double> GetScoreAsync(string peerId);
    Task UpdateAsync(string peerId, double delta);
}

// IModelAdapter lives in the peer domain (peer/src/ModelAdapter).
// Defined here for cross-reference; orchestrator does NOT use it directly.
interface IModelAdapter
{
    // Execute a completion request against the configured model backend.
    Task<ModelAdapterResult> CompleteAsync(ModelAdapterRequest request, CancellationToken ct = default);

    // Check if the adapter can serve requests (model loaded and reachable).
    Task<bool> IsAvailableAsync(CancellationToken ct = default);
}

record ModelAdapterRequest(
    string Prompt,
    int MaxTokens,
    string? SystemPrompt = null);

record ModelAdapterResult(
    string Completion,
    double Confidence,    // [0.0, 1.0] — estimated by adapter or model
    int PromptTokens,
    int CompletionTokens);
```

## 8. Error Handling

| Situation | Action |
|---|---|
| Peer timeout on `/infer` | Select backup peer; re-issue with same `task_id` |
| Signature invalid on response | Reject; apply negative reputation delta (-0.1) |
| No peers available for role | Fail session task; return `peer_unavailable` |
| ConsensusEngine returns no winner | Fail task; log `internal_error` |
| Session not found | Return `session_not_found` |
| All critics unavailable | Accept highest-confidence answer without scoring (log warning) |

## 9. Configuration

```yaml
orchestrator:
  reputation_threshold: -0.5
  liveness_window_seconds: 30
  max_task_retries: 2
  consensus:
    algorithm: weighted-majority   # weighted-majority | bft (v0.2)
  dht:
    bootstrap_address: "host:port"
```

## 10. Hand-off to Sprint 5

Sprint 5 (peer design) will consume:
- Peer API contracts from docs/api-peer.md
- `IDhtClient` interface (for DhtPublisher in peer node)
- `IModelAdapter` interface (defined in this sprint, see section 7)
- `IConsensusEngine` — NOT needed in peer; consensus lives in orchestrator only
