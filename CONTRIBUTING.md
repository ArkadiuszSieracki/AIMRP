# Contributing to AIMRP

Thank you for your interest in AIMRP.

## Scope

| Area | Path | Review path |
|---|---|---|
| RFC normative text | `docs/rfc/` | RFC change process (below) |
| Reference implementation | `peer/`, `orchestrator/`, `proto/` | Standard PR review |
| Tooling and CI | `.github/`, `scripts/` | Standard PR review |
| Documentation | `docs/` (non-RFC) | Standard PR review |

## Developer Certificate of Origin (DCO)

All commits MUST be signed off using the [Developer Certificate of Origin 1.1](https://developercertificate.org/).

Add `Signed-off-by: Your Name <your.email@example.com>` to every commit message. Use `git commit -s` to add this automatically. By signing off you certify that you wrote the contribution or otherwise have the right to submit it under the project's licenses.

We do **not** require a separate CLA. The DCO is sufficient.

## Pull Request Process

1. Fork and create a topic branch: `feat/<area>-<short-desc>` or `fix/<area>-<short-desc>`.
2. Commit using [Conventional Commits](https://www.conventionalcommits.org/):
   - `feat(rfc): add §11.14 …`
   - `fix(peer): correct AAC verification edge case`
   - `docs(readme): clarify MCB phase`
3. Test locally. For RFC changes, run the linter defined in Appendix G of the RFC.
4. Open a PR against `main`. Link related issues.
5. At least one maintainer approval is required (see `GOVERNANCE.md`).

## RFC Change Process

Changes to `docs/rfc/RFC-AIMRP-0.1.md` follow a stricter process:

1. Open an issue with `rfc-change` label describing problem, proposed change, and impact (additive / breaking).
2. Discussion period: minimum 7 days for additive changes, 14 days for breaking changes.
3. PR referencing the issue. Breaking changes require approval from 2+ maintainers.
4. Linter pass per Appendix G is mandatory.
5. Schema changes require corresponding updates in Appendix D and §20.2 message type registry.

## Code of Conduct

By participating you agree to abide by `CODE_OF_CONDUCT.md`.

## Licensing

By contributing, you agree that:
- Code contributions are licensed under Apache License 2.0 (`LICENSE`).
- Documentation contributions (including RFC text) are licensed under CC BY 4.0 (`LICENSE.docs`).

## Questions

Open a GitHub Discussion or issue with the `question` label.

---

© 2026 Arkadiusz Sieracki and AIMRP Contributors. Licensed under CC BY 4.0 (see `LICENSE.docs`).
