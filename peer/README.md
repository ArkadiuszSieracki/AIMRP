# AIMRP Peer Node — Design Document

Version: 0.1  
Sprint: 5

## 1. Overview

A peer node is a network participant in the AIMRP protocol. It:
- Generates and holds an Ed25519 identity keypair
- Publishes a signed manifest to the DHT
- Serves the Peer API (GET /capabilities, POST /plan, POST /infer, POST /score)
- Executes reasoning tasks via a configurable model backend
- Participates in sessions as one or more roles (planner, reasoner, critic, retriever)

The peer does NOT run consensus logic — that belongs to the orchestrator.

## 2. Module Diagram

```
PeerNode (entry point)
  │
  ├── ApiServer                 — HTTP server; routes requests to handlers
  │     ├── CapabilitiesHandler
  │     ├── PlanHandler
  │     ├── InferHandler
  │     └── ScoreHandler
  │
  ├── CapabilitiesProvider      — reads config; returns roles + model list
  │
  ├── IModelAdapter             — model execution abstraction
  │     ├── OpenAiCompatibleAdapter   — Ollama / LM Studio / vLLM / OpenRouter / OpenAI / Azure
  │     └── AnthropicAdapter          — Claude (separate API shape)
  │
  ├── DhtPublisher              — signs and publishes manifest; runs refresh loop
  │     └── uses: IDhtClient
  │
  └── SignatureService          — Ed25519 sign / verify
```

## 3. Startup Lifecycle

```
LOAD config (peer.yaml or env vars)
  │
  ▼
LOAD or GENERATE Ed25519 keypair
  │  keypair stored at: config.identity.key_path
  │  peer_id = SHA-256(pubkey) hex-encoded
  ▼
BUILD manifest
  │  roles, models, endpoints from config
  │  timestamp = now, nonce = random UUID
  │  sign manifest with private key
  ▼
DhtPublisher.JoinAsync(bootstrap_address)
  │  connect to DHT network
  ▼
DhtPublisher.PublishAsync(manifest)
  │  write signed manifest to DHT
  ▼
ApiServer.StartAsync()
  │  begin serving on configured address
  ▼
DhtPublisher: start refresh loop (every TTL/2 seconds)
  │
  ▼
PEER IS OPERATIONAL
```

## 4. Shutdown Lifecycle

```
SIGTERM / cancellation received
  │
  ▼
ApiServer.StopAsync()     — stop accepting new requests; drain in-flight
  │
  ▼
DhtPublisher.LeaveAsync() — withdraw manifest from DHT
  │
  ▼
DISPOSE all resources
```

## 5. IModelAdapter: Config-Driven Backend Selection

The model backend is selected entirely by configuration. Zero code change required to switch.

### 5.1 OpenAiCompatibleAdapter

Covers: Ollama, LM Studio, llama.cpp server, vLLM, OpenRouter, Together.ai, OpenAI, Azure OpenAI.

All these expose `POST /v1/chat/completions` (OpenAI Chat Completions format).

Request shape sent by adapter:
```json
{
  "model": "<config.model.name>",
  "messages": [
    { "role": "system", "content": "<system_prompt>" },
    { "role": "user",   "content": "<prompt>" }
  ],
  "max_tokens": <max_tokens>,
  "temperature": 0.7
}
```

Response fields consumed:
- `choices[0].message.content` → Completion
- `usage.prompt_tokens` → PromptTokens
- `usage.completion_tokens` → CompletionTokens
- `confidence` → not in OpenAI spec; adapter computes heuristic from `logprobs` if available, else returns 0.75

### 5.2 AnthropicAdapter

Covers: Claude 3.x (claude-3-5-sonnet, claude-3-opus, etc.)

Uses Anthropic Messages API: `POST https://api.anthropic.com/v1/messages`

Request shape:
```json
{
  "model": "<config.model.name>",
  "max_tokens": <max_tokens>,
  "system": "<system_prompt>",
  "messages": [
    { "role": "user", "content": "<prompt>" }
  ]
}
```

Response fields consumed:
- `content[0].text` → Completion
- `usage.input_tokens` → PromptTokens
- `usage.output_tokens` → CompletionTokens

### 5.3 Interface

```csharp
interface IModelAdapter
{
    Task<ModelAdapterResult> CompleteAsync(ModelAdapterRequest request, CancellationToken ct = default);
    Task<bool> IsAvailableAsync(CancellationToken ct = default);
}

record ModelAdapterRequest(
    string Prompt,
    int MaxTokens,
    string? SystemPrompt = null);

record ModelAdapterResult(
    string Completion,
    double Confidence,
    int PromptTokens,
    int CompletionTokens);
```

### 5.4 Adapter Selection by Config

```yaml
model_adapter:
  type: openai-compatible    # or: anthropic
  endpoint: http://localhost:11434/v1   # Ollama local
  model: llama3.2:3b
  api_key: ""                # empty for local; set for cloud APIs
```

To switch to large cloud model — change config only:
```yaml
model_adapter:
  type: openai-compatible
  endpoint: https://api.openai.com/v1
  model: gpt-4o
  api_key: "${OPENAI_API_KEY}"
```

## 6. Endpoint Handlers

### GET /capabilities
- Calls `CapabilitiesProvider.GetCapabilities()`
- Returns peer_id, roles, models from config
- No model adapter call required

### POST /plan
- Only valid if peer advertises role PLANNER
- Calls `IModelAdapter.CompleteAsync()` with structured planning prompt
- Parses LLM output into ordered `TaskStep` list
- Signs PlanResponse before returning

### POST /infer
- Valid for roles: REASONER, RETRIEVER
- Calls `IModelAdapter.CompleteAsync()` with task prompt
- Wraps result in InferResponse with confidence and signature

### POST /score
- Only valid if peer advertises role CRITIC
- Calls `IModelAdapter.CompleteAsync()` with scoring prompt per criterion
- Parses scores from LLM output (JSON extraction)
- Returns ScoreResponse with per-criterion scores and overall mean

## 7. Signature Service

All critical responses are signed with the peer's Ed25519 private key.

```csharp
interface ISignatureService
{
    // Sign the canonical JSON serialization of the payload.
    byte[] Sign(object payload);

    // Verify a signature against the signer's public key.
    bool Verify(object payload, byte[] signature, byte[] pubkey);
}
```

Canonical serialization: UTF-8 JSON, keys sorted, no whitespace, excluding the `signature` field itself.

.NET implementation: `System.Security.Cryptography.ECDiffieHellman` is NOT used — use `NSec.Cryptography` or `BouncyCastle` for Ed25519.

## 8. DhtPublisher

```csharp
interface IDhtPublisher
{
    // Build, sign, and publish the peer's manifest. Called on startup and on refresh.
    Task PublishAsync(CancellationToken ct = default);

    // Start background refresh loop (publishes every TTL/2 seconds).
    Task StartRefreshLoopAsync(CancellationToken ct = default);

    // Withdraw manifest from DHT. Called on shutdown.
    Task LeaveAsync(CancellationToken ct = default);
}
```

## 9. Security Rules

- Private key MUST NOT be logged or transmitted.
- All inbound requests MUST include `X-AIMRP-Version` header; reject with 400 if missing or unsupported.
- Signature on outbound responses MUST be computed over final serialized payload (after all fields are set).
- API key for model backend MUST come from env var or secrets manager — never hardcoded in config file.
- Prompts received via `/infer` MUST be treated as untrusted input — do not execute or eval them.

## 10. Configuration Schema

```yaml
peer:
  identity:
    key_path: "./peer-key.pem"         # Ed25519 keypair file

  network:
    listen_address: "0.0.0.0:8080"
    public_address: "my-peer-host:8080"

  roles:
    - planner
    - reasoner

  model_adapter:
    type: openai-compatible            # openai-compatible | anthropic
    endpoint: http://localhost:11434/v1
    model: llama3.2:3b
    api_key: ""

  dht:
    bootstrap_address: "bootstrap-node:4000"
    ttl_seconds: 3600

  liveness:
    heartbeat_interval_seconds: 60
```

## 11. Hand-off to Sprint 6

Sprint 6 (DX and CLI) will consume:
- `peer.yaml` config schema (section 10) — CLI commands will generate or validate this
- Startup lifecycle (section 3) — quickstart guide describes this sequence
- `GET /capabilities` — CLI uses it to inspect a live peer
- `IModelAdapter` adapter types — "How to implement your own peer" guide covers this
