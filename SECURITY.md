# Security Policy

AIMRP is a P2P protocol with cryptographic, economic, and consensus attack surfaces. We follow a **90-day coordinated disclosure** model.

## Reporting a Vulnerability

**Do NOT open public GitHub issues for security vulnerabilities.**

Use GitHub private vulnerability reporting via the **Security** tab of this repository. Email contact will be added once `aimrp.org` DNS is delegated.

PGP key: TBD (will be published before v1.0).

Include in your report:
- Affected component (RFC section, file path, or implementation module)
- Reproduction steps or proof-of-concept
- Suggested CVSS 3.1 score (we will reassess)
- Whether you intend to publish independently after embargo

## Disclosure Timeline

| Day | Action |
|-----|--------|
| 0 | Report received; acknowledgement within 72 hours |
| 0–14 | Triage, severity assessment (CVSS 3.1), reproduction |
| 14–75 | Patch development, peer review, embargoed testing |
| 75–90 | Pre-disclosure to known operators (≥7 days notice) |
| 90 | Public advisory + CVE published; patch released |

Extensions beyond 90 days require mutual agreement. If we fail to respond within 14 days, you MAY proceed to public disclosure.

## Severity and Patch SLA

| CVSS 3.1 | Severity | Patch SLA |
|---|---|---|
| 9.0–10.0 | Critical | 7 days |
| 7.0–8.9 | High | 30 days |
| 4.0–6.9 | Medium | 60 days |
| 0.1–3.9 | Low | 90 days |

## Scope

In-scope:
- Protocol design flaws in `docs/rfc/RFC-AIMRP-0.1.md`
- Reference implementation bugs (when published)
- Cryptographic weaknesses (Ed25519 misuse, JCS edge cases, PQ migration per §11.13)
- AAC forgery, quorum bypass, VRF manipulation
- DHT attacks (Sybil, eclipse, partition exploitation per §11.12.2)
- Wallet rotation race conditions (§11.12.3)
- MCB atomicity violations (§7.6)
- Compliance level bypass (§21)

Out-of-scope:
- Third-party deployments and forks
- Social engineering of maintainers
- DoS via legitimate protocol traffic (operator concern, see §11.10 `rate_limit_exceeded`)
- Issues in dependencies that have their own disclosure process

## Safe Harbor

Good-faith security research conducted in accordance with this policy will not result in legal action from the AIMRP project. We support CVE assignment via MITRE and will credit reporters in the published advisory unless they request anonymity.

This policy is aligned with §22 (Security Considerations) of the AIMRP RFC.

---

© 2026 Arkadiusz Sieracki and AIMRP Contributors. Licensed under CC BY 4.0 (see `LICENSE.docs`).
