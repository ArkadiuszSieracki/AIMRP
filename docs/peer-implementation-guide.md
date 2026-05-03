# How to Implement Your Own AIMRP Peer

Version: 0.1.0  
Sprint: 6

This guide walks through building a minimal AIMRP-compatible peer node. It assumes .NET 8+ and access to at least one model backend (Ollama locally or any OpenAI-compatible API).

## 0. Minimal Peer Architecture

```
          ┌──────────────────────────────┐
  HTTP ─▶│ ApiHost (Kestrel / ASP.NET)  │◄── X-AIMRP-Version validation
          │   │ /capabilities             │
          │   │ /plan  /infer  /score    │
          └───┬───────────────────────────┘
              ▼
  ┌─────────────────┐   ┌───────────────────────┐
  │ RoleHandlers   │─▶│ IModelAdapter         │─▶ Ollama / OpenAI / vLLM
  │  Planner       │   │   CompleteAsync       │
  │  Reasoner      │   │   IsAvailableAsync    │
  │  Critic        │   └───────────────────────┘
  │  Retriever     │
  └───┬─────────────┘
      ▼
  ┌────────────────┐         ┌────────────────┐
  │ Signer (Ed25519)│─────────▶│ IDhtClient    │──▶ Kademlia DHT
  └────────────────┘         │  Publish      │
                            │  Lookup       │
                            │  Join / Leave │
                            └────────────────┘
```

Key separations:
- **ApiHost** owns transport concerns (headers, content-type, body limits).
- **RoleHandlers** own per-role prompt assembly and response shaping.
- **IModelAdapter** is the only place that knows about the backend (Ollama, OpenAI, etc.).
- **Signer** is invoked exactly once per outbound signed payload (manifest, plan, infer, score).
- **IDhtClient** isolates Kademlia mechanics from the rest of the peer.

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

### 9.1 Minimal IModelAdapter Implementation (OpenAI-compatible)

```csharp
public sealed class OpenAiCompatibleAdapter : IModelAdapter
{
    private readonly HttpClient _http;
    private readonly string _model;

    public OpenAiCompatibleAdapter(HttpClient http, string endpoint, string model, string? apiKey)
    {
        _http = http;
        _http.BaseAddress = new Uri(endpoint.TrimEnd('/') + "/");
        _model = model;
        if (!string.IsNullOrEmpty(apiKey))
        {
            _http.DefaultRequestHeaders.Authorization =
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", apiKey);
        }
    }

    public async Task<ModelAdapterResult> CompleteAsync(
        ModelAdapterRequest request, CancellationToken ct = default)
    {
        var body = new
        {
            model       = string.IsNullOrEmpty(request.ModelName) ? _model : request.ModelName,
            messages    = new[]
            {
                new { role = "system", content = request.SystemPrompt ?? string.Empty },
                new { role = "user",   content = request.Prompt }
            },
            max_tokens  = request.MaxTokens,
            temperature = request.Temperature ?? 0.7,
            top_p       = request.TopP       ?? 1.0,
            stop        = request.Stop?.ToArray() ?? Array.Empty<string>()
        };

        using var resp = await _http.PostAsJsonAsync("chat/completions", body, ct);
        resp.EnsureSuccessStatusCode();

        using var stream = await resp.Content.ReadAsStreamAsync(ct);
        using var doc    = await JsonDocument.ParseAsync(stream, cancellationToken: ct);

        var root       = doc.RootElement;
        var completion = root.GetProperty("choices")[0].GetProperty("message").GetProperty("content").GetString() ?? "";
        var usage      = root.TryGetProperty("usage", out var u)
            ? new UsageStats(u.GetProperty("prompt_tokens").GetUInt32(), u.GetProperty("completion_tokens").GetUInt32())
            : new UsageStats(0, 0);

        return new ModelAdapterResult(completion, usage, ConfidenceSource: "heuristic", Confidence: 0.75, ModelUsed: body.model);
    }

    public async Task<bool> IsAvailableAsync(CancellationToken ct = default)
    {
        try
        {
            using var resp = await _http.GetAsync("models", ct);
            return resp.IsSuccessStatusCode;
        }
        catch { return false; }
    }
}
```

This is intentionally minimal. Production peers SHOULD also: extract logprobs when the backend exposes them, surface `model_error` on parse failures, and apply per-call timeouts.

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

## 12. Common Pitfalls

| # | Pitfall | Symptom | Fix |
|---|---|---|---|
| 1 | Signing the **non-canonical** payload | Other peers reject your manifest with `signature_invalid` | Apply RFC §3.2.1 canonical JSON (sorted keys, NFC, no whitespace) **before** Ed25519 sign. |
| 2 | Reusing `nonce` across refreshes | DHT validators drop refresh as replay | Generate a fresh UUID v4 on every publish/refresh. |
| 3 | Logging the private key during init | Catastrophic key disclosure | Never log `Span<byte>` for private key material; redact at log boundary. |
| 4 | Returning `confidence` outside [0.0, 1.0] | Orchestrator weighting blows up | Clamp the output of `exp(mean(logprob))`; fall back to 0.75 when logprobs absent. |
| 5 | Ignoring `X-AIMRP-Version` / `AIMRP-Version` | Cross-version traffic reaches business logic | Validate header in middleware; return `version_unsupported` (HTTP 400) for any mismatch. |
| 6 | Sharing a single `HttpClient` per request | Socket exhaustion under load | Use `IHttpClientFactory` or a long-lived singleton per backend. |
| 7 | Treating `/infer.prompt` as trusted | Prompt injection / shell exec / SSRF | Only pass `prompt` to the model adapter; never to `Process.Start`, eval, or unconstrained URL fetch. |
| 8 | Hardcoding the bootstrap node IP | Peer becomes undiscoverable when bootstrap rotates | Allow multiple `bootstrap_nodes`, support DNS SRV (`_aimrp-dht._tcp.<domain>`). |
| 9 | Crashing on transient model errors | Peer enters reboot loop, manifest expires | Catch in `IModelAdapter.CompleteAsync`; return `model_error` and stay RUNNING/DEGRADED. |
| 10 | Forgetting to drain on SIGTERM | Orchestrators see truncated responses | Stop accepting new requests, wait for in-flight, **then** `DhtClient.LeaveAsync()`. |
