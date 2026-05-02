# Appendix B: Threat Matrix (STRIDE)

This appendix categorizes threats to AIMRP using a STRIDE-style model.

| Category | Threat | Description | Impact | Mitigation |
|---------|--------|-------------|---------|------------|
| S — Spoofing | Identity Forgery | Attacker pretends to be another peer | Consensus corruption | Mandatory signatures, peer_id = pubkey hash |
| S — Spoofing | Orchestrator Impersonation | Fake orchestrator sends tasks | Prompt abuse, poisoning | Local validation of all tasks |
| T — Tampering | DHT Record Injection | Fake manifests in DHT | Routing manipulation | Signed manifests, rate limiting |
| T — Tampering | Task Manipulation | Altering tasks in transit | Wrong reasoning | Transport encryption |
| R — Repudiation | Fake Results | Peer denies producing output | Accountability loss | Signed task results |
| I — Information Disclosure | Prompt Exfiltration | Peer leaks prompts or outputs | Privacy breach | No persistence, sandbox |
| I — Information Disclosure | Host Data Exfiltration | Prompt injection to read files | Catastrophic | Mandatory sandbox, filtering |
| D — Denial of Service | DHT Flooding | Overloading routing tables | Network slowdown | Rate limiting, quotas |
| D — Denial of Service | Task Flooding | Sending thousands of tasks | Peer overload | Local rate limiting |
| E — Elevation of Privilege | Prompt-based Privilege Escalation | “Encrypt all files”, “Run shell” | Host compromise | No system access, filtering |
| E — Elevation of Privilege | Model Manipulation | Poisoning planning or scoring | Consensus corruption | Multi-role validation |
