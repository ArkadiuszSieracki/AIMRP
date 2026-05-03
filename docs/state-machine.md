# AIMRP State Machines

Version: 0.1.0  
Sprint: 6+

## 1. Peer State Machine

### States

```
OFFLINE → STARTING → OPERATIONAL → DRAINING → OFFLINE
                         │
                         ├── SESSION_ACTIVE (per session)
                         │       │
                         │       ├── TASK_IDLE
                         │       ├── TASK_ASSIGNED
                         │       ├── TASK_EXECUTING
                         │       └── TASK_COMPLETED
                         │
                         └── ERROR (recoverable)
```

### State Transitions

| From | Event | To | Action |
|---|---|---|---|
| OFFLINE | startup | STARTING | Load config, load keypair |
| STARTING | DHT join success | OPERATIONAL | Publish manifest; start API server |
| STARTING | DHT join failure | OFFLINE | Log error; retry with backoff |
| OPERATIONAL | SIGTERM received | DRAINING | Stop accepting new requests |
| DRAINING | in-flight requests drained | OFFLINE | Leave DHT; shutdown |
| OPERATIONAL | SESSION_INIT received | SESSION_ACTIVE | Validate session_id; allocate slot |
| SESSION_ACTIVE | TASK_ASSIGN received | TASK_ASSIGNED | Validate task; queue for execution |
| TASK_ASSIGNED | model inference starts | TASK_EXECUTING | Call IModelAdapter |
| TASK_EXECUTING | inference complete | TASK_COMPLETED | Sign result; return TASK_RESULT |
| TASK_EXECUTING | model error | TASK_IDLE | Return `model_error`; slot available |
| TASK_EXECUTING | safety filter reject | TASK_IDLE | Return `unsafe_prompt`; slot available |
| TASK_COMPLETED | session ends | OPERATIONAL | Release session slot |
| OPERATIONAL | model backend unreachable | ERROR | Return `peer_unavailable` on new requests |
| ERROR | model backend recovers | OPERATIONAL | Resume normal operation |

### Invariants

- A peer in `DRAINING` MUST NOT accept new SESSION_INIT messages.
- A peer MUST NOT execute more concurrent tasks than `max_concurrent_tasks` (default: 4).
- A peer in `ERROR` MAY continue to serve `GET /capabilities` (advertising limited availability).

---

## 2. Orchestrator State Machine

### States

```
IDLE → PLANNING → EXECUTING → SCORING → CONSENSUS → DONE
                                                        │
                                                        └── FAILED (any phase)
```

### State Transitions

| From | Event | To | Action |
|---|---|---|---|
| IDLE | client request received | PLANNING | Select planner peer; send TASK_ASSIGN |
| PLANNING | TASK_RESULT (steps) received | EXECUTING | Distribute task steps to reasoner peers |
| PLANNING | timeout or error | FAILED | Return error to client |
| EXECUTING | all TASK_RESULTS received | SCORING | Assign results to critic peers |
| EXECUTING | partial results + timeout | SCORING | Proceed with available results; log missing |
| SCORING | all scores received | CONSENSUS | Run consensus algorithm |
| SCORING | timeout | CONSENSUS | Proceed with available scores |
| CONSENSUS | winner selected | DONE | Return result to client; update reputation |
| CONSENSUS | no winner (tie or quorum failure) | FAILED | Return `internal_error` to client |
| DONE | — | IDLE | Session closed |
| FAILED | — | IDLE | Session closed; error logged |

### Session Timeout Policy

- Each phase has a hard timeout (configurable, default: 60s per phase).
- If the total session timeout (`session_hard_timeout`, default: 600s) is reached in any state → transition to FAILED immediately.

---

## 3. Session Protocol Flow

```
Client          Orchestrator         Planner Peer      Reasoner Peer     Critic Peer
  │                  │                    │                  │                │
  │──── request ────►│                    │                  │                │
  │                  │─── POST /plan ────►│ (plan task)      │                │
  │                  │◄── TASK_RESULT ───┐│                  │                │
  │                  │                    (steps list)       │                │
  │                  │─── POST /infer ──────────────────────►│ (exec step)    │
  │                  │◄── TASK_RESULT ────────────────────┐  │ (answer)       │
  │                  │─── POST /score ────────────────────────────────────── ►│
  │                  │◄── SCORE_RESULT ───────────────────────────────────── ──│
  │                  │  [run consensus]   │                  │                │
  │◄── response ─────│                    │                  │                │
```
