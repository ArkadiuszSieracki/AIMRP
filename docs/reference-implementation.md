# AIMRP Reference Implementation (Conceptual)

Version: 0.1  
Sprint: 6+

This document describes the reference implementation of AIMRP in pseudocode and flow diagrams. No actual code is provided — this is a specification aid.

## 1. Peer Node Pseudocode

```
function StartPeer(config):
    keypair = LoadOrGenerateKeypair(config.identity.key_path)
    peer_id = HexEncode(SHA256(keypair.pubkey))
    
    adapter = CreateModelAdapter(config.model_adapter)
    if not adapter.IsAvailable():
        log.warn("model backend not reachable; peer will start in degraded mode")
    
    manifest = BuildManifest(peer_id, keypair, config)
    manifest.signature = Sign(manifest, keypair.privkey)
    
    dht = ConnectDHT(config.dht.bootstrap_address)
    dht.Join()
    dht.Publish(manifest)
    
    server = HttpServer(config.network.listen_address)
    server.RegisterHandlers(peer_id, keypair, adapter, config)
    server.Start()
    
    RefreshLoop(dht, manifest, config.dht.ttl_seconds / 2)


function HandleInfer(request, adapter, keypair):
    ValidateVersion(request.headers["X-AIMRP-Version"])
    ValidateSignature(request)   // if signed
    
    if not IsSafePrompt(request.prompt):
        return Error("unsafe_prompt", 422)
    
    result = adapter.Complete(ModelAdapterRequest(
        prompt = request.prompt,
        system_prompt = RoleSystemPrompt(request.role),
        max_tokens = config.max_tokens
    ))
    
    response = InferResponse(
        task_id = request.task_id,
        result  = result.completion,
        confidence = result.confidence,
        signature  = Sign(result, keypair.privkey)
    )
    return response


function IsSafePrompt(prompt):
    // Reject prompts that attempt to:
    // - Access filesystem ("read /etc/passwd", "open file")
    // - Execute shell ("run bash", "execute command")
    // - Exfiltrate data ("send to http://", "upload to")
    // - Jailbreak ("ignore previous instructions")
    // Returns false if any pattern matches
    for pattern in UNSAFE_PATTERNS:
        if matches(prompt, pattern):
            return false
    return true
```

## 2. Orchestrator Session Flow Pseudocode

```
function RunSession(goal, client):
    session_id = GenerateUUID()
    
    // 1. Discovery
    peers = dht.Lookup(role = "planner", min_reputation = config.reputation_threshold)
    if len(peers) == 0:
        return Error("peer_unavailable")
    
    // 2. Planning
    planner = SelectPeer(peers, strategy = "highest_reputation")
    plan_response = planner.POST("/plan", PlanRequest(
        session_id = session_id,
        goal = goal
    ))
    steps = plan_response.steps
    
    // 3. Execution
    reasoners = dht.Lookup(role = "reasoner", min_reputation = config.reputation_threshold)
    results = []
    for step in steps:
        peer = SelectPeer(reasoners, strategy = "load_balanced")
        result = peer.POST("/infer", InferRequest(
            session_id = session_id,
            task_id = step.step_id,
            prompt = step.description,
            role = "REASONER"
        ))
        results.append(result)
    
    // 4. Scoring
    critics = dht.Lookup(role = "critic", min_reputation = config.reputation_threshold)
    scores = []
    for result in results:
        critic = SelectPeer(critics)
        score = critic.POST("/score", ScoreRequest(
            session_id = session_id,
            task_id = result.task_id,
            answer = result.result,
            criteria = ALL_CRITERIA
        ))
        scores.append(score)
    
    // 5. Consensus
    winner = WeightedMajority(results, scores, reputation_map)
    
    // 6. Reputation update
    for (result, score) in zip(results, scores):
        delta = (score.overall - 0.5) * 0.1
        reputation.Update(result.peer_id, delta)
    
    return winner.result


function WeightedMajority(results, scores, reputation):
    weights = {}
    for result in results:
        score = scores[result.task_id]
        rep = reputation[result.peer_id]  // normalized to [0, 1]
        weight = score.overall * rep * result.confidence
        answer = Normalize(result.result)  // canonical form for comparison
        weights[answer] = weights.get(answer, 0) + weight
    
    return argmax(weights)
```

## 3. ModelAdapter Pseudocode

```
// OpenAI-compatible adapter
function CompleteOpenAI(request, config):
    body = {
        "model": config.model,
        "messages": [
            {"role": "system", "content": request.system_prompt},
            {"role": "user",   "content": request.prompt}
        ],
        "max_tokens": request.max_tokens,
        "temperature": 0.7
    }
    
    response = HTTP.POST(
        url     = config.endpoint + "/chat/completions",
        headers = {"Authorization": "Bearer " + config.api_key},
        body    = body,
        timeout = 120s
    )
    
    completion = response.choices[0].message.content
    confidence = EstimateConfidence(response)  // from logprobs if available; else 0.75
    
    return ModelAdapterResult(
        completion        = completion,
        confidence        = confidence,
        prompt_tokens     = response.usage.prompt_tokens,
        completion_tokens = response.usage.completion_tokens
    )


// Anthropic adapter
function CompleteAnthropic(request, config):
    body = {
        "model":      config.model,
        "max_tokens": request.max_tokens,
        "system":     request.system_prompt,
        "messages":   [{"role": "user", "content": request.prompt}]
    }
    
    response = HTTP.POST(
        url     = "https://api.anthropic.com/v1/messages",
        headers = {
            "x-api-key": config.api_key,
            "anthropic-version": "2023-06-01"
        },
        body    = body,
        timeout = 120s
    )
    
    return ModelAdapterResult(
        completion        = response.content[0].text,
        confidence        = 0.75,  // Anthropic does not expose logprobs
        prompt_tokens     = response.usage.input_tokens,
        completion_tokens = response.usage.output_tokens
    )
```

## 4. Safety Filter Pseudocode

```
UNSAFE_PATTERNS = [
    // Filesystem access
    r"(read|open|write|delete|list)\s+(file|/|\.\.)",
    r"(cat|ls|rm|cp|mv|chmod)\s+",
    // Shell execution
    r"(exec|execute|run|spawn|bash|sh|cmd|powershell)",
    r"subprocess|shell=True|os\.system",
    // Network exfiltration
    r"(send|upload|post|http://|https://|ftp://)\s+.*(data|result|content)",
    // Jailbreak patterns
    r"ignore (previous|above|all) instructions",
    r"you are now|pretend you are|act as if",
    r"DAN|do anything now",
]

function IsSafePrompt(prompt):
    normalized = prompt.lower().strip()
    for pattern in UNSAFE_PATTERNS:
        if regex_match(pattern, normalized):
            log.warn("unsafe prompt detected", pattern=pattern)
            return false
    return true
```

## 5. Key Derivation

```
function DerivePeerId(pubkey_bytes):
    // pubkey_bytes: raw Ed25519 public key (32 bytes)
    digest = SHA256(pubkey_bytes)         // 32 bytes
    peer_id = hex_encode(digest)          // 64 hex characters, lowercase
    // Example: "a3f9c2d1e5b8f4a7c0d3e6b9f2a5c8d1e4b7f0a3c6d9e2b5f8a1c4d7e0b3f6a9"
    return peer_id
    // NOTE: no "sha256-" prefix. The prefix is NOT part of the protocol.
```
