# AIMRP Consensus Design

Version: 0.1  
Sprint: 4

## 1. Purpose

This document describes the consensus strategy for AIMRP:
- v0.1 execution algorithm (weighted majority)
- v0.2 strategic target (BFT)
- Algorithm comparison and selection rationale

## 2. v0.1: Weighted Majority + Reputation

### Design Goals

- Simple to implement and debug
- Resistant to individual low-quality outputs
- Reward peers with consistent quality

### Algorithm

Given a set of candidate answers for a single task:

```
for each candidate c:
    reputation_normalized(c) = (reputation_score(c.peer_id) + 1) / 2   // [0, 1]
    critic_score(c)          = mean of all critic overall scores for c   // [0, 1]
    weight(c)                = critic_score(c) × reputation_normalized(c) × c.confidence

winner = argmax weight(c)
```

### Failure Modes

| Scenario | Behavior |
|---|---|
| All critics unavailable | Fall back to highest raw confidence |
| Single candidate only | Accept without consensus scoring |
| All weights equal | Select by highest raw confidence; tie-break by peer_id |
| Winner weight < threshold | Return result with low-confidence warning |

### Limitations

- Does not tolerate Byzantine (actively malicious) peers
- A high-reputation peer can dominate if critics are absent
- No finality guarantee — orchestrator decision is unilateral

## 3. v0.2 Target: Byzantine Fault Tolerance (BFT)

### Motivation

When the peer network is public and permissionless, v0.1 is insufficient.
A BFT protocol is required to tolerate up to f malicious peers in a network of 3f+1 peers.

### Algorithm Candidates

#### PBFT (Practical Byzantine Fault Tolerance)

- Classic 3-phase protocol: pre-prepare → prepare → commit
- Finality: strong (2f+1 agreement required)
- Latency: O(n²) message complexity — suitable for small peer sets (< 20)
- Implementation: relatively straightforward for .NET

#### HotStuff

- Pipeline-based BFT with O(n) message complexity
- Used in: Meta's Diem/LibraBFT, Aptos
- Better latency at scale
- More complex to implement correctly

#### Tendermint

- Round-based BFT with leader rotation
- Well-specified, widely deployed (Cosmos ecosystem)
- Good fit if peer discovery is already DHT-based

### Decision Matrix

| Criterion | PBFT | HotStuff | Tendermint |
|---|---|---|---|
| Implementation complexity | Medium | High | Medium |
| Message complexity | O(n²) | O(n) | O(n) |
| Finality | Strong | Strong | Strong |
| Suitable peer count | < 20 | < 100 | < 100 |
| .NET library availability | None (custom) | None (custom) | None (custom) |
| Specification clarity | High | High | High |

**Recommendation for v0.2:** Start with **PBFT** for initial BFT implementation.
Rationale: best-documented protocol, strong finality, manageable for the initial permissionless deployment scale.
Migrate to HotStuff if the peer network grows beyond 20 active peers per session.

## 4. Interface Contract

Both v0.1 and v0.2 implement the same interface:

```csharp
interface IConsensusEngine
{
    Task<ConsensusResult> ResolveAsync(
        IReadOnlyList<CandidateAnswer> candidates,
        CancellationToken ct = default);
}

record CandidateAnswer(
    string PeerId,
    string Completion,
    double Confidence,
    IReadOnlyList<CriticScore> Scores);

record CriticScore(
    string CriticPeerId,
    double Overall,
    IReadOnlyDictionary<string, double> PerCriterion);

record ConsensusResult(
    string WinningCompletion,
    double ConsensusScore,
    string WinnerPeerId,
    bool IsLowConfidence);
```

## 5. Minimum Peer Requirements per Algorithm

| Algorithm | Min peers for 1 faulty peer | Min peers for f faulty |
|---|---|---|
| Weighted majority (v0.1) | 1 (no fault tolerance) | N/A |
| PBFT (v0.2) | 4 | 3f+1 |
| HotStuff (v0.2+) | 4 | 3f+1 |

## 6. Open Items

- Final BFT algorithm selection: PBFT (recommended) vs HotStuff
- Quorum size configuration (k peers per session)
- View-change protocol handling for PBFT (leader failure)
- Integration with reputation system in BFT mode
