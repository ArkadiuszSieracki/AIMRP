# AIMRP Operational Guidelines

Version: 0.1.0  
Sprint: 6+

Recommendations for deploying AIMRP peers and orchestrators in production environments.

## 1. Hardware Recommendations

### Minimum (Class L Peer — single reasoner role, small model)

| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 20 GB SSD | 50 GB SSD |
| Network | 10 Mbps | 100 Mbps |
| GPU | Not required | Optional (improves inference speed) |

### Recommended (Class F/S Peer — multiple roles, mid-size model)

| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 8 cores | 16 cores |
| RAM | 32 GB | 64 GB |
| Disk | 100 GB SSD | 200 GB NVMe |
| Network | 100 Mbps | 1 Gbps |
| GPU | 8 GB VRAM | 16–24 GB VRAM |

### Orchestrator

| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 4 cores | 8 cores |
| RAM | 4 GB | 16 GB |
| Disk | 10 GB SSD | 50 GB SSD |
| Network | 100 Mbps | 1 Gbps |

## 2. Recommended Model Selection by Role

| Role | Local (dev) | Production (quality) | Production (speed) |
|---|---|---|---|
| planner | llama3.2:3b | gpt-4o / claude-3-5-sonnet | gpt-4o-mini |
| reasoner | llama3.2:3b | gpt-4o / claude-3-5-sonnet | mistral-7b-instruct |
| critic | llama3.2:3b | gpt-4o / claude-3-opus | gpt-4o-mini |
| retriever | nomic-embed-text | text-embedding-3-large | text-embedding-3-small |

**Guidance:**
- For critic role, prefer models with strong instruction-following and structured output (JSON).
- For planner role, prefer models with large context windows (≥32k tokens).
- Never assign a model smaller than 7B parameters to the critic role in production — scoring quality degrades significantly.

## 3. Model Sandbox Configuration

All model backends MUST be isolated:

| Control | Requirement | Implementation |
|---|---|---|
| Filesystem access | NONE | Run model in container with read-only FS or no volume mounts |
| Network access | Only model API endpoint | Network policy / firewall rules |
| Shell execution | NONE | No `subprocess`, `exec`, `shell=True` |
| Memory limit | Set explicitly | Container memory limit |
| CPU limit | Set explicitly | Container CPU quota |

Recommended sandbox: Docker/Podman container with `--network=none` for the model process, or a VM with strict network policy.

## 4. Reputation Settings

| Parameter | Default | Notes |
|---|---|---|
| `reputation_threshold` | -0.5 | Exclude peers below this score from routing |
| `reputation_delta_scale` | 0.1 | Magnitude of per-task reputation update |
| `reputation_warmup_tasks` | 5 | Minimum tasks before reputation affects routing |
| `reputation_reset_on_keychange` | true | New keypair = new identity = score 0 |

Conservative setting for untrusted public networks: set `reputation_threshold` to 0.0 (only allow peers with net positive history).

## 5. Timeout Configuration

| Parameter | Default | Notes |
|---|---|---|
| Model inference timeout | 120s | Increase for large models / complex tasks |
| Peer liveness check timeout | 30s | Reduce for latency-sensitive applications |
| Session hard timeout | 600s | Total session duration limit |
| Task retry delay | 2s | Backoff between retries |
| DHT manifest TTL | 3600s | Reduce to 600s for highly dynamic networks |

## 6. Monitoring and Observability

Peers and orchestrators SHOULD expose the following metrics:

| Metric | Description |
|---|---|
| `tasks_total` | Counter: total tasks received |
| `tasks_completed` | Counter: tasks completed successfully |
| `tasks_failed` | Counter: tasks failed (by error code) |
| `inference_duration_seconds` | Histogram: model inference latency |
| `reputation_score` | Gauge: current reputation score (for reporting to orchestrator) |
| `dht_publish_latency_seconds` | Histogram: DHT manifest publish time |
| `active_sessions` | Gauge: current active sessions |
| `rate_limited_requests_total` | Counter: requests rejected due to rate limiting |

Recommended export format: Prometheus `/metrics` endpoint.

## 7. Security Hardening (Production)

- Rotate Ed25519 keypair only intentionally (reputation resets with new key).
- Store `api_key` in a secrets manager (Vault, AWS Secrets Manager, etc.) — never in config files.
- Enable TLS 1.3 on all peer-to-peer and client-to-orchestrator connections.
- Deploy behind a reverse proxy (nginx, Caddy) for TLS termination and IP rate limiting.
- Set `max_concurrent_tasks: 2` for public-facing peers to reduce DoS exposure.
- Enable request logging with prompt content DISABLED (log metadata only).
