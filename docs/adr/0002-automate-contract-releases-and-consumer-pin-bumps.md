# ADR-0002: Automate contract releases and consumer pin-bumps

## Status

Proposed <!-- Proposed | Accepted | Deprecated | Superseded by [ADR-XXXX](XXXX-filename.md) -->

## Date

2026-08-06

## Context

[ADR-0001](0001-version-the-contract-with-semver-and-a-deprecate-dont-break-policy.md)
set the versioning and compatibility policy, but *enacting* it is currently
manual: a human edits `info.version` in four documents, hand-writes a
changelog, cuts a tag, and then edits the pinned `SPEC_VERSION` in each
consumer ([`datocal.com`](https://git.datolab.com), GitLab; `macrolibre.app`,
GitHub). Manual version numbering under schedule pressure is exactly where a
wrong bump or a missed pin-bump slips in — the drift risk
[ADR-0003](../../../datocal.com/docs/adr/0003-runtime-safety-and-data-integrity-strategy.md)
and [ADR-0004](0004-publish-the-public-api-contract-from-a-dedicated-spec-repo.md)
both flag.

The commercial product (`datocal.com`, Pro/B2B) leads the roadmap and release
cadence — the open contract *follows* business decisions about which endpoints
ship and when. Automation should let that cadence move fast without giving up
the safety `ADR-0001` guarantees.

Two layers need automating, and they automate differently:

- **Layer A — releasing the contract**: choosing the semver bump, generating a
  changelog, and cutting the tag in this repo.
- **Layer B — propagating the pin**: updating `SPEC_VERSION` in each consumer
  once a new contract version exists.

## Decision Drivers

- Automate as much as possible without losing `ADR-0001`'s two-party gate on
  breaking (major) changes
- The bump for an *API contract* should be derived from what actually changed
  in the contract, not from human-authored commit messages
- Do not re-introduce the coupling `ADR-0004` avoided — the contract repo must
  not need to know about, or hold credentials for, its consumers
- Work across hosts: this repo and `macrolibre.app` are GitHub; `datocal.com`
  is GitLab
- Every adopted change must still pass through the consumer's contract test

## Options Considered

### Layer A — how the contract is released

**Auto-tag on merge (semantic-release style).** Every merge to `main` that
looks releasable is tagged and published automatically.

- **Good:** Zero-touch; fastest cadence.
- **Bad:** No human gate on a *public* contract others pin — removes
  `ADR-0001`'s two-party sign-off on majors. Relies on commit-message
  discipline to classify breaking changes, which humans get wrong.

**Release-PR, oasdiff-driven bump (chosen).** On changes to `main`, a pipeline
diffs the contract against the last tag with `oasdiff`, derives the semver bump
(breaking → major; additive → minor; else patch), generates a changelog from
the diff (`oasdiff changelog`), and opens a "Release vX.Y.Z" PR. A human merges
it to cut the release.

- **Good:** The bump is derived from the *actual* API delta, not commit
  messages — authoritative for a contract. The changelog describes real
  contract changes. Merging the release PR is the natural home for
  `ADR-0001`'s major-change sign-off. Reuses the `oasdiff` tooling already
  installed for the compatibility gate.
- **Bad:** A human still merges each release (by design). Release tooling
  differs by host, though for this GitHub repo it is native.

### Layer B — how consumers adopt a new pin

**Manual pin-bump PRs (`ADR-0004`'s status quo).** A person edits
`SPEC_VERSION` in each consumer after a release.

- **Good:** Simplest; no tooling.
- **Bad:** Easy to forget, leaving a consumer silently on an old contract —
  the exact failure `ADR-0004` flagged.

**Contract repo pushes PRs into consumers (dispatch).** The release pipeline
opens MRs/PRs in the consumers via their APIs.

- **Good:** Fully automatic propagation.
- **Bad:** Couples the contract to its consumers — it needs their identities
  and cross-host credentials (a GitHub Action writing to GitLab). Re-creates
  exactly the coupling `ADR-0004` rejected.

**Renovate in each consumer, pull-based (chosen).** Each consumer runs Renovate
with a custom manager that tracks the `SPEC_VERSION` pin against this repo's
GitHub tags. A new tag → Renovate opens the bump MR (GitLab) / PR (GitHub),
which runs the consumer's contract test.

- **Good:** Fully automatic *and* decoupled — the contract repo knows nothing
  about consumers; each opts in and pulls. Works on both hosts. The bump lands
  as a normal reviewed+CI'd MR/PR, so the contract test gates every adoption.
  Auto-merge can be scoped by change type to preserve governance.
- **Bad:** Per-consumer Renovate config to maintain; auto-merge safety depends
  on how thorough the consumer's contract test is.

## Decision

**Layer A — release-PR, oasdiff-driven.** A workflow in this repo diffs `main`
against the latest tag with `oasdiff`, derives the semver bump, regenerates the
changelog, updates all four `info.version` in lockstep, and opens a release PR.
Merging it cuts and pushes the tag. Majors carry `ADR-0001`'s two-party
sign-off at that merge.

**Layer B — Renovate, pull-based, governance- and phase-gated auto-merge.**
Each consumer runs Renovate tracking `SPEC_VERSION` against this repo's
`github-tags`. Adoption policy:

- **major** → never auto-merge; requires the two-party review (`ADR-0001`).
- **minor / patch** → auto-merge on a green contract test **only from `1.0.0`,
  and only once the consumer's `ADR-0003` decoder contract tests are real**.
- **During `0.x`, and while contract tests are still thin** → review-only for
  every bump, because a `0.y` change can break (`ADR-0001`) and a thin test
  cannot yet catch it.

## Consequences

### Positive

- No hand-numbered versions; the bump reflects the real contract diff
- Changelog is generated from actual API changes, not commit prose
- Consumers are notified and PR'd automatically, closing the "forgot to bump"
  drift risk — without the contract repo touching them
- `ADR-0001`'s governance is preserved and even strengthened (the gate is now
  mechanical), letting the commercial cadence lead safely
- Reuses `oasdiff`; no bespoke diffing

### Negative / Trade-offs

- Release tooling and Renovate config differ per host (GitHub vs GitLab) and
  per consumer
- Auto-merge is only as safe as the consumer's contract test — it stays off
  until those tests are meaningful
- A human still merges each contract release PR (intended, not incidental)

### Risks

- Renovate classifies a `0.y` bump as "minor" though `0.x` permits breaking —
  mitigated by the phase gate (no auto-merge during `0.x`)
- Auto-merge could rubber-stamp a breaking change a thin contract test misses —
  mitigated by keeping auto-merge off until `ADR-0003` decoder tests exist
- Cross-host release tooling can drift in behavior — keep the authoritative
  bump/changelog logic in `oasdiff`, which is host-agnostic

## Links

- Builds on: [ADR-0001](0001-version-the-contract-with-semver-and-a-deprecate-dont-break-policy.md)
  (the policy this automates)
- Relates to: [ADR-0004](0004-publish-the-public-api-contract-from-a-dedicated-spec-repo.md)
  (upgrades its manual pin-bump flow without re-introducing coupling)
- Relates to: `datocal.com`
  [ADR-0003](../../../datocal.com/docs/adr/0003-runtime-safety-and-data-integrity-strategy.md)
  (the decoder contract tests that make auto-merge safe)
- Reference: `oasdiff` `changelog` / `breaking` commands; Renovate custom
  managers (`github-tags` datasource)
