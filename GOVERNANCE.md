# AIMRP Project Governance

This document describes how decisions are made in the AIMRP project. It complements `CONTRIBUTING.md` and is referenced by §24 of the AIMRP RFC.

## Project Status

AIMRP is currently in **early specification stage** (v0.1 RC1). The project is led by its original author and is open to additional maintainers as the community grows.

## Roles

### Maintainers

Maintainers have commit and merge rights. Initial maintainer:

- **Arkadiusz Sieracki** — original author, RFC editor

New maintainers are added by lazy consensus of existing maintainers after a sustained pattern of high-quality contributions (typically 3+ months and 5+ merged non-trivial PRs).

### Contributors

Anyone who submits a PR, opens an issue, or participates in discussion. No formal status; recognized in `CHANGELOG.md` and release notes.

### RFC Editor

A maintainer designated to maintain consistency of the RFC document. Currently Arkadiusz Sieracki. The RFC editor has final say on RFC text style and structure (not on normative content, which follows the RFC change process).

## Decision Making

### Lazy Consensus

Most decisions are made by lazy consensus: a proposal is announced (via PR, issue, or discussion), and if no maintainer objects within the comment window, it is considered approved.

Comment windows:
- Standard PR: 72 hours
- RFC change (additive): 7 days
- RFC change (breaking): 14 days, requires 2+ maintainer approvals
- New maintainer nomination: 14 days
- Governance change: 30 days

### Objection and Resolution

Any maintainer MAY object during the comment window. Objections SHOULD include rationale and a proposed alternative. Resolution path:

1. Discussion in the PR/issue thread.
2. If unresolved within 7 days, escalate to a synchronous maintainer meeting (recorded summary published).
3. If still unresolved, the RFC editor (for RFC matters) or project lead (for other matters) makes a decision, with rationale documented.

## Code Review Standards

- All PRs require at least one maintainer approval before merge.
- Authors MUST NOT merge their own PRs without a co-maintainer approval (when more than one maintainer exists).
- CI MUST pass (including RFC linter for `docs/rfc/` changes).
- Breaking changes require a `BREAKING CHANGE:` footer per Conventional Commits and explicit version bump.

## Releases

- RFC versions follow `MAJOR.MINOR` (currently `0.1`).
- Reference implementation versions follow [SemVer](https://semver.org/).
- Release notes published in `CHANGELOG.md` (when implementation begins).

## Trademark and Branding

The name "AIMRP" and associated logos are claimed by the original author. Use in derivative or unrelated projects requires attribution per `NOTICE` and SHOULD avoid implying endorsement.

## Amendments

Changes to this document follow the Governance change process above (30-day comment window, all maintainers must approve).

---

© 2026 Arkadiusz Sieracki and AIMRP Contributors. Licensed under CC BY 4.0 (see `LICENSE.docs`).
