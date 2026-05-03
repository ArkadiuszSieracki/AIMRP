# AIMRP Peer API Reference

Version: 0.1.0  
Transport: HTTP/JSON-first (gRPC binding defined in proto/aimrp.proto)

## Common Rules

- All requests and responses MUST include header `X-AIMRP-Version: 0.1`.
- All responses on error MUST include an error envelope (see RFC section 11b).
- `session_id` and `task_id` are UUIDs (string format, e.g. `"550e8400-e29b-41d4-a716-446655440000"`).
- `role` in `/infer` is a direct top-level string field (not nested in metadata).
- `signature` fields are base64-encoded Ed25519 signatures of the serialized payload.
- `confidence` and `score` values are floats in range [0.0, 1.0].

## Endpoints

### GET /capabilities

Returns the peer's identity, supported roles, and available models.

No request body required.

**Response 200:**

```json
{
  "peer_id": "abc123...",
  "roles": ["planner", "reasoner"],
  "models": [
    {
      "name": "llama-3.2-3b",
      "max_context": 8192,
      "max_tokens": 1024,
      "avg_latency_ms": 180
    }
  ]
}
```

**Errors:** `peer_unavailable`, `internal_error`

---

### POST /plan

Asks the peer (acting as planner) to decompose a problem into ordered reasoning steps.

**Request:**

```json
{
  "session_id": "uuid",
  "problem": "Compare RAFT and PBFT consensus algorithms for a semi-trusted AI peer network.",
  "context": "Optional background text or retrieved facts."
}
```

**Response 200:**

```json
{
  "session_id": "uuid",
  "steps": [
    {
      "step_id": "uuid",
      "description": "Summarize RAFT consensus properties",
      "suggested_role": "reasoner"
    },
    {
      "step_id": "uuid",
      "description": "Summarize PBFT consensus properties",
      "suggested_role": "reasoner"
    },
    {
      "step_id": "uuid",
      "description": "Compare and recommend",
      "suggested_role": "critic"
    }
  ],
  "signature": "base64"
}
```

**Errors:** `invalid_request`, `session_not_found`, `model_error`, `peer_unavailable`

---

### POST /infer

Executes a single reasoning task for the given session. The peer uses its model to produce a completion.

**Request:**

```json
{
  "session_id": "uuid",
  "task_id": "uuid",
  "prompt": "Summarize the key properties of the RAFT consensus algorithm.",
  "max_tokens": 512,
  "role": "reasoner",
  "model_name": "llama-3.2-3b",
  "temperature": 0.2,
  "top_p": 0.95,
  "stop": ["\n\n"]
}
```

**Optional sampling fields:**

| Field | Type | Range | Default | Notes |
|---|---|---|---|---|
| `model_name` | string | — | peer default | Pin a specific model from `/capabilities`. Peer MUST return `invalid_request` if not advertised. |
| `temperature` | number | [0.0, 2.0] | 0.7 | `0.0` = greedy. Peers MAY clamp to backend-supported range. |
| `top_p` | number | (0.0, 1.0] | 1.0 | Nucleus sampling. Mutually combinable with `temperature`. |
| `stop` | string[] | up to 4 | `[]` | Stop sequences passed to the backend verbatim. |

Peers MUST silently ignore unknown sampling parameters (forward compatibility). Peers MUST NOT fail a request because an optional sampling field is absent.

**Response 200:**

```json
{
  "task_id": "uuid",
  "completion": "RAFT is a leader-based consensus algorithm designed for understandability...",
  "confidence": 0.87,
  "usage": {
    "prompt_tokens": 134,
    "completion_tokens": 312
  },
  "signature": "base64"
}
```

**Errors:** `invalid_request`, `session_not_found`, `task_not_found`, `model_error`, `peer_unavailable`, `rate_limited`

---

### POST /score

Asks the peer (acting as critic) to score a candidate answer according to defined criteria.

**Request:**

```json
{
  "session_id": "uuid",
  "task_id": "uuid",
  "answer": "RAFT is a leader-based consensus algorithm...",
  "criteria": ["logic", "factuality", "completeness"]
}
```

**Criteria values:** `logic`, `factuality`, `relevance`, `completeness`, `safety`

**Response 200:**

```json
{
  "task_id": "uuid",
  "scores": {
    "logic": 0.92,
    "factuality": 0.85,
    "completeness": 0.78
  },
  "overall": 0.85,
  "signature": "base64"
}
```

**Errors:** `invalid_request`, `session_not_found`, `task_not_found`, `model_error`, `peer_unavailable`

**Error example (HTTP 400):**

```json
{
  "error": {
    "code": "invalid_request",
    "message": "role not supported: this peer does not advertise 'critic'",
    "details": {
      "peer_id": "a3f9c2...",
      "advertised_roles": "reasoner,planner"
    }
  }
}
```

---

## Error Response Shape

```json
{
  "error": {
    "code": "model_error",
    "message": "Model returned an empty completion.",
    "details": {}
  }
}
```

See RFC section 11b for the full error code list.

## Endpoint Address Format

Peer endpoints use structured objects in manifests:

```json
{ "type": "http-json", "address": "host:port" }
```

Type `http-json` is the v0.1 primary binding. Type `grpc` is reserved for future use.

## Model Selection Rules

When `/infer` receives a request:

1. **Explicit selection.** If `model_name` is present and matches an entry in `/capabilities.models[].name`, the peer MUST execute on that model.
2. **No match.** If `model_name` is present and does NOT match any advertised model, return `invalid_request` with `details.model_name` echoed.
3. **Default.** If `model_name` is empty/absent, the peer MUST use the model marked as default in its config (or the first entry in `models[]` when no explicit default is configured).
4. **Role compatibility.** Even when `model_name` matches, the peer MUST verify the model is suitable for the requested `role` (e.g. structured output for `planner`/`critic`). On mismatch return `model_error` with `details.reason = "model_role_incompatible"`.
5. **Multi-model peers.** Peers MAY advertise multiple models; orchestrator-side `PeerFilter.model_name_hint` performs substring matching against `models[].name` (see DHT design §5.2).
6. **Echo.** The response MUST set `model_name` to the resolved model so the orchestrator can attribute usage and reputation correctly.
