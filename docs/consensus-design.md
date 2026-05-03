# AIMRP Consensus Design

Version: 0.1.0  
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

**Low-confidence threshold (normative):** The default `low_confidence_threshold` is **0.35** on the final winning weight (after critic×reputation×confidence multiplication). Orchestrators MAY override per-deployment within `[0.20, 0.60]`. When the winning weight falls below the threshold the `ConsensusResult.IsLowConfidence` flag MUST be `true` and `SessionResult.scored` MUST still report the actual evaluation outcome. AIMRP-Strict deployments SHOULD additionally fail the task with `internal_error` when below threshold AND fewer than 2 critics participated.

### Weighted Majority Flow

```
   reasoner peers              critic peers              reputation store
        │                          │                          │
        │ (peer_id, completion,    │ (per_criterion + overall) │
        │  confidence)             │                          │
        ▼                          ▼                          ▼
   ┌────────────────────────────────────────────────────┐
   │  for each candidate c:                                  │
   │     critic_score(c)        = mean(critic_overall)       │
   │     reputation_norm(c)     = (rep + 1) / 2              │
   │     weight(c)              = critic_score × rep × conf  │
   └────────────────────────────────┬────────────────────────┘
                                  ▼
                       ┌────────────────────┐
                       │   argmax weight     │
                       └──────────┬──────────┘
                                  ▼
               IsLowConfidence = winner.weight < 0.35 ?
                                  ▼
                            ConsensusResult
```

### Worked Numerical Example

Three reasoners produce candidates for the same task; two critics score each.

| Candidate | Reasoner rep | Confidence | Critic1 overall | Critic2 overall |
|---|---|---|---|---|
| A | 0.60 | 0.85 | 0.90 | 0.84 |
| B | 0.10 | 0.92 | 0.70 | 0.66 |
| C | -0.20 | 0.78 | 0.55 | 0.60 |

Compute:

```
critic_score(A) = (0.90 + 0.84) / 2 = 0.870
rep_norm(A)     = (0.60 + 1) / 2     = 0.800
weight(A)       = 0.870 × 0.800 × 0.85 = 0.5916

critic_score(B) = (0.70 + 0.66) / 2 = 0.680
rep_norm(B)     = (0.10 + 1) / 2     = 0.550
weight(B)       = 0.680 × 0.550 × 0.92 = 0.3441

critic_score(C) = (0.55 + 0.60) / 2 = 0.575
rep_norm(C)     = (-0.20 + 1) / 2    = 0.400
weight(C)       = 0.575 × 0.400 × 0.78 = 0.1794
```

Winner = **A** (weight 0.5916). `IsLowConfidence = false` (≥ 0.35).

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

### PBFT Flow (3-phase)

```
client → leader      pre-prepare           prepare           commit
  │         │  ----------------▶   ----------------▶  ----------------▶
  │         │  (leader assigns         (replicas echo       (replicas confirm
  │         │   sequence + sig)         pre-prepare)          2f+1 prepares seen)
  │         │
  │   replicas (n = 3f+1)
  │         R1   R2   R3   ...   Rn
  │         ▲    ▲    ▲          ▲
  │         │    │    │          │
  │         broadcast at every phase
  │
  ▼
 REPLY (collect f+1 matching replies → commit accepted)
```

Finality is reached when any client observes `f+1` matching `REPLY` messages. View-change handles leader failure; details deferred to v0.2 implementation spec.

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

## 7. Reputation in BFT Mode

In v0.2 BFT mode the deterministic agreement on the *winning answer* is provided by the BFT protocol; reputation no longer decides correctness, only **eligibility and weighting of the candidate set**.

Normative rules:

1. **Quorum admission.** Only peers with `reputation_score ≥ reputation_admission_threshold` (default `0.0`) MAY be sampled into a quorum.
2. **Weighted vote (optional).** When the BFT protocol supports weighted votes (e.g. PBFT with stake), each replica's vote weight MUST be `max(0, reputation_score)`. Peers with negative reputation MUST NOT be admitted.
3. **Post-commit reputation update.** After commit, replicas that voted with the committed value receive `+Δ`; replicas that diverged receive `−Δ` regardless of view-change outcome.
4. **No retroactive demotion.** A reputation drop MUST NOT invalidate an already-committed BFT decision.
5. **VRF-based sampling** (RFC §7.4) selects the actual quorum from the eligible set; reputation only filters eligibility.
