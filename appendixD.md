# Appendix D: Secure Implementation Checklist

## D.1 Identity & Authentication
- [ ] Peer generates Ed25519 keypair
- [ ] peer_id = hash(pubkey)
- [ ] All messages signed
- [ ] All signatures verified

## D.2 Transport Security
- [ ] QUIC or TLS 1.3
- [ ] Replay protection enabled
- [ ] Integrity protection enabled

## D.3 Prompt Safety
- [ ] No filesystem access
- [ ] No shell access
- [ ] No network access
- [ ] Prompt filtering enabled
- [ ] Unsafe prompts return unsafe_prompt

## D.4 DHT Safety
- [ ] Manifests signed
- [ ] Rate limiting enabled
- [ ] Schema validation enabled

## D.5 Consensus Safety
- [ ] Weighted consensus
- [ ] Reputation integrated
- [ ] Outlier detection enabled

## D.6 Privacy
- [ ] No prompt storage
- [ ] No output storage
- [ ] No session logs
- [ ] Reputation stores only metadata

## D.7 Hardening
- [ ] Sandboxed model execution
- [ ] Resource limits (CPU, RAM)
- [ ] Task rate limiting
- [ ] Timeout enforcement
