# AIMRP Session Flow Examples

This document provides concrete flow examples for AIMRP session execution.
All examples assume HTTP/JSON transport and signed critical messages.
All HTTP requests and responses are expected to include `X-AIMRP-Version: 0.1`.

## 1. Example A: Happy Path (Single Task)

### Scenario

- Session asks for short architectural recommendation.
- One planner, two reasoners, one critic.

### Flow

1. Orchestrator creates `session_id`.
2. Orchestrator calls planner `/plan`.
3. Planner returns one step: "Compare two architecture options".
4. Orchestrator assigns same task to two reasoners via `/infer`.
5. Reasoners return candidate answers + confidence.
6. Orchestrator sends both answers to critic via `/score`.
7. Critic scores logic/factuality.
8. `ConsensusEngine` computes weighted result using critic score and peer reputation.
9. Orchestrator returns final answer and updates reputation.

### Outcome

- Session completed in one round.
- Reputation increases for aligned high-quality outputs.

## 2. Example B: Multi-Step Session with Retriever

### Scenario

- Query requires staged reasoning with external context retrieval.

### Flow

1. Planner returns three steps:
   - Retrieve relevant facts (`retriever`)
   - Synthesize reasoning (`reasoner`)
   - Validate assumptions (`critic`)
2. Orchestrator assigns retrieval task to retriever peer.
3. Retriever returns context snippets and source metadata.
4. Orchestrator injects retrieved context into reasoner `/infer` prompt.
5. Reasoner returns synthesized answer.
6. Critic scores answer and identifies weak assumptions.
7. If score below threshold, orchestrator opens refinement iteration.
8. Final consensus picks best validated output.

### Outcome

- Better answer quality through structured role decomposition.

## 3. Example C: Peer Failure and Fallback

### Scenario

- Selected reasoner times out.

### Flow

1. Orchestrator dispatches `/infer` with timeout budget.
2. Timeout expires.
3. Orchestrator marks peer attempt as failed for this task.
4. Orchestrator selects backup peer from DHT lookup with same role.
5. Task is re-issued with same `task_id` and incremented attempt metadata.
6. Session proceeds if minimum quorum is still satisfied.

### Outcome

- Session remains available despite individual peer failure.

## 4. Example D: Suspected Malicious Output

### Scenario

- A peer returns high-confidence but low-quality output inconsistent with others.

### Flow

1. Critic score is low.
2. Majority agreement is low for that output.
3. Consensus rejects candidate despite high self-reported confidence.
4. Reputation delta is negative for outlier behavior.
5. If repeated, peer falls below routing threshold.

### Outcome

- Protocol resists single-peer manipulation via scoring + reputation.

## 5. Example E: BFT-Ready Transition (Roadmap)

### Scenario

- Deployment evolves to permissionless network with stronger adversarial assumptions.

### Flow (target shape)

1. Session result candidates enter BFT consensus phases.
2. Validators exchange prepare/commit votes.
3. Finalization requires supermajority.
4. Final answer accepted only after protocol finality.

### Outcome

- Higher trust in adversarial environments at cost of latency and complexity.

## 6. Minimal Payload Skeletons

### 6.1 Inference Request

```json
{
  "session_id": "uuid",
  "task_id": "uuid",
  "prompt": "...",
  "role": "reasoner"
}
```

### 6.2 Inference Result

```json
{
  "task_id": "uuid",
  "completion": "...",
  "confidence": 0.77,
  "signature": "base64"
}
```

### 6.3 Score Result

```json
{
  "task_id": "uuid",
  "scores": { "logic": 0.86, "factuality": 0.81 },
  "overall": 0.84,
  "signature": "base64"
}
```

## 7. Implementation Notes for Sprint 2

- Convert payload skeletons into canonical schema (`proto/aimrp.proto` and/or JSON schema).
- Define retry policy and idempotency semantics for `task_id`.
- Standardize error envelopes for timeout, signature failure, and validation errors.
