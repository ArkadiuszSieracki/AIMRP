# How to Implement Your Own AIMRP Peer

Version: 0.1  
Sprint: 6

This guide walks through building a minimal AIMRP-compatible peer node. It assumes .NET 8+ and access to at least one model backend (Ollama locally or any OpenAI-compatible API).

## 1. Prerequisites

- .NET 8 SDK
- A running model backend:
  - **Local (recommended for dev):** [Ollama](https://ollama.com) with a small model: `ollama pull llama3.2:3b`
  - **Cloud:** OpenAI API key, or any OpenAI-compatible endpoint
- Access to a DHT bootstrap node (`localhost:4000` for local dev)

## 2. Generate a Keypair

```bash
aimrp peer keygen --out ./peer-key.pem
```

This creates an Ed25519 keypair. The public key hash is your `peer_id`.

Your `peer_id` is permanent as long as you keep the same keypair. Losing the keypair means losing your reputation score.

## 3. Write a Config File

Create `peer.yaml`:

```yaml
peer:
  identity:
    key_path: "./peer-key.pem"

  network:
    listen_address: "0.0.0.0:8080"
    public_address: "localhost:8080"   # reachable address for other peers

  roles:
    - reasoner                          # start with one role; add more as needed

  model_adapter:
    type: openai-compatible
    endpoint: http://localhost:11434/v1  # Ollama
    model: llama3.2:3b
    api_key: ""

  dht:
    bootstrap_address: "localhost:4000"
    ttl_seconds: 3600

  liveness:
    heartbeat_interval_seconds: 60
```

Validate the config:

```bash
aimrp config validate --file peer.yaml --type peer
```

## 4. Choose Your Role(s)

| Role | What your peer does | Model requirement |
|---|---|---|
| `planner` | Decomposes a goal into task steps | Structured output (list/JSON) |
| `reasoner` | Executes a task step and returns an answer | General instruction following |
| `critic` | Scores answers by criterion | Structured scoring output |
| `retriever` | Fetches and synthesizes context | Embedding or retrieval synthesis |

A peer can serve multiple roles. Start with `reasoner` — it has the broadest model compatibility.

## 5. Choose Your Model Backend

### Option A — Local (Ollama)

```yaml
model_adapter:
  type: openai-compatible
  endpoint: http://localhost:11434/v1
  model: llama3.2:3b
  api_key: ""
```

No API key needed. Ollama serves `POST /v1/chat/completions` natively.

### Option B — OpenAI

```yaml
model_adapter:
  type: openai-compatible
  endpoint: https://api.openai.com/v1
  model: gpt-4o
  api_key: "${OPENAI_API_KEY}"
```

Set `OPENAI_API_KEY` as an environment variable. Never hardcode the key.

### Option C — Anthropic Claude

```yaml
model_adapter:
  type: anthropic
  endpoint: https://api.anthropic.com/v1/messages
  model: claude-3-5-sonnet-20241022
  api_key: "${ANTHROPIC_API_KEY}"
```

### Option D — Other OpenAI-compatible (vLLM, LM Studio, OpenRouter, Azure OpenAI)

```yaml
model_adapter:
  type: openai-compatible
  endpoint: https://openrouter.ai/api/v1
  model: mistralai/mistral-7b-instruct
  api_key: "${OPENROUTER_API_KEY}"
```

Any endpoint that serves `POST /v1/chat/completions` works with `type: openai-compatible`.

## 6. Start the Peer

```bash
aimrp peer start --config peer.yaml
```

Expected startup output:
```
[INFO] keypair loaded: peer_id=a3f9c2...
[INFO] manifest built: roles=[reasoner], model=llama3.2:3b
[INFO] DHT joined: bootstrap=localhost:4000
[INFO] manifest published: TTL=3600s
[INFO] API server listening on 0.0.0.0:8080
[INFO] peer operational
```

## 7. Verify Your Peer

From another terminal, call your peer's capabilities endpoint:

```bash
aimrp network inspect --address localhost:8080
```

Or directly:

```bash
curl -H "X-AIMRP-Version: 0.1" http://localhost:8080/capabilities
```

Expected response:

```json
{
  "peer_id": "a3f9c2...",
  "roles": ["reasoner"],
  "models": [{ "name": "llama3.2:3b", "family": "llama", "context_window": 128000 }],
  "endpoints": [{ "type": "http", "address": "localhost:8080" }],
  "protocol_version": "0.1"
}
```

## 8. Required HTTP Headers

Every request to your peer MUST include:

```
X-AIMRP-Version: 0.1
```

Your peer MUST return this header in every response. Requests missing the header receive HTTP 400:

```json
{
  "error": {
    "code": "version_unsupported",
    "message": "X-AIMRP-Version header missing or unsupported"
  }
}
```

## 9. Implement IModelAdapter (for .NET implementations)

```csharp
interface IModelAdapter
{
    Task<ModelAdapterResult> CompleteAsync(ModelAdapterRequest request, CancellationToken ct = default);
    Task<bool> IsAvailableAsync(CancellationToken ct = default);
}
```

For OpenAI-compatible backends, send:

```json
POST /v1/chat/completions
{
  "model": "<your model>",
  "messages": [
    { "role": "system", "content": "<system_prompt>" },
    { "role": "user",   "content": "<prompt>" }
  ],
  "max_tokens": 1024
}
```

Read `choices[0].message.content` as the completion.

## 10. Security Checklist

Before connecting to a shared network:

- [ ] Ed25519 keypair generated and stored securely
- [ ] Private key never logged or transmitted
- [ ] `api_key` set via environment variable, not in config file
- [ ] `X-AIMRP-Version` header validated on all inbound requests
- [ ] Prompts from `/infer` treated as untrusted input (no eval, no shell exec)
- [ ] Config file does not contain secrets in plain text

## 11. Next Steps

- Add more roles (planner, critic) by extending your `roles:` list and updating prompt templates
- Join a shared DHT to discover other peers: update `dht.bootstrap_address`
- Monitor your reputation score via the orchestrator's peer listing
- Read the full RFC: [docs/rfc/RFC-AIMRP-0.1.md](rfc/RFC-AIMRP-0.1.md)
