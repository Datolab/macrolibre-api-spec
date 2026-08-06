# ADR-0001: Version the contract with semver and a deprecate-don't-break policy

## Status

Proposed <!-- Proposed | Accepted | Deprecated | Superseded by [ADR-XXXX](XXXX-filename.md) -->

## Date

2026-08-06

## Context

This repo is the public OpenAPI contract that `datocal.com` (private
backend) and `macrolibre.app` (public client) both pin a specific tagged
version of, per ADR-0004. B2B integrators (FR-F-4) also build against it
without access to either implementation. The four documents (Epic B/C/E/F)
now have real endpoints and schemas, so the contract needs a versioning
scheme and a compatibility policy before anything downstream pins to it.

Two questions are open:

1. **How is a version number assigned and what does it mean?** All four
   documents currently carry a placeholder `info.version: 0.1.0` that was
   never released, tagged, or consumed.
2. **How does the contract change over time without breaking the consumers
   that have pinned it?** ADR-0004 explicitly left the governance of a
   breaking change undecided and flagged it as a risk.

A consumer of an API contract needs exactly one thing from a version number:
a reliable signal of whether adopting a newer version is safe. That is what
this decision must provide.

Note on scope: this ADR versions **the contract itself** (this repo's git
tag and each document's `info.version`). It is a distinct axis from the
**dataset releases** of Epic C, which are calendar-versioned `vYYYY.MM`
(FR-C-1) and describe data payloads, not the wire contract. The two must not
be conflated.

## Decision Drivers

- Consumers must be able to reason about compatibility from the version alone
- A pinned consumer must never be broken by a change within the version line
  it pinned — the strongest guarantee an API contract can offer
- Breaking changes should be rare, deliberate, and heavily gated; additive
  changes should be cheap
- The scheme must express pre-stable maturity (this contract is early)
- Should resolve ADR-0004's open "who approves a breaking change" question
  rather than leave it dangling

## Options Considered

### Option 1: Semver + phased 0.x + additive/deprecate-in-place (PHP-style)

Semantic Versioning for the contract, one repo-wide version (single git tag,
all four `info.version` in lockstep). Pre-stable maturity is expressed via
`0.x` per semver's "major version zero is for initial development": `0.0.x`
alpha, `0.1.x`+ beta, `1.0.0` the first stable contract. After `1.0.0`,
changes follow a **deprecate-don't-break** lifecycle modelled on PHP's
backward-compatibility policy: never remove or change the shape of an existing
element within a major; instead add the replacement alongside, mark the old
one `deprecated`, and remove it only at the next major after a sunset window.

- **Good:** A pinned consumer is never broken within a major — the strongest
  guarantee. Uses only native OpenAPI 3.1 mechanisms (`deprecated: true` on
  operations, parameters, and schema properties; `x-` extensions for
  `since`/replacement; `Deprecation`/`Sunset` response headers per RFC 8594).
  Additive changes stay cheap (minor); only removals cost a major. Governance
  falls out naturally by change type (see Decision). Semver is the lingua
  franca every consumer and tool already understands.
- **Bad:** Requires discipline — teams must add-alongside instead of editing
  in place, and carry deprecated elements until a major. A little more surface
  area lives in the spec during the deprecation window.

### Option 2: URL / path major versioning (`/v1`, `/v2` in parallel)

Encode the major version in the path and run multiple majors simultaneously.

- **Good:** A hard, visible boundary between incompatible versions; clients
  migrate on their own schedule.
- **Bad:** The backend must implement and operate every live major in
  parallel — real cost for a small team. Doesn't remove the need for a
  compatibility policy *within* a major; it just adds an orthogonal, heavier
  mechanism on top. Overkill for a contract this young.

### Option 3: Calendar versioning or no formal policy

Version the contract by date (like the dataset releases), or leave
compatibility to ad-hoc judgement.

- **Good:** Trivial to assign.
- **Bad:** A date carries no compatibility signal — a consumer cannot tell a
  breaking change from a fix. Conflates the contract's version axis with the
  dataset-release calendar axis. Fails the one job the version has.

## Decision

**Option 1.** The contract is versioned with **Semantic Versioning**, as a
single repo-wide version: one git tag, with all four documents'
`info.version` kept identical to it.

Maturity phases:

| Phase  | Version   | Meaning                                                        |
|--------|-----------|---------------------------------------------------------------|
| Alpha  | `0.0.x`   | Shape still moving; may break freely                          |
| Beta   | `0.1.x`+  | Stabilizing; `0.y` bump = breaking, `0.y.z` = additive/fix    |
| Stable | `1.0.0`   | Contract frozen under the compatibility policy below          |

The contract's current maturity is **alpha**, so `info.version` is set to
`0.0.1` (correcting the unreleased `0.1.0` placeholder).

Compatibility policy (binding from `1.0.0`; applied as a goal during `0.x`):

- **Never break a pinned consumer within a major version.**
- A change that would be breaking is instead made **additive**: introduce the
  new endpoint/field/shape alongside the existing one (**minor** bump).
- The superseded element is marked `deprecated: true` with `x-deprecated-since`
  and a pointer to its replacement (**minor** bump — metadata only, still
  compatible). At runtime the server SHOULD emit `Deprecation` and `Sunset`
  (RFC 8594) headers for deprecated elements.
- A deprecated element is **removed only at the next major**, no earlier than
  its published sunset date.

Governance (resolving ADR-0004's open question) is gated by change type:

- **minor / patch** (additive or deprecation — cannot break a consumer):
  lightweight approval, Datolab alone.
- **major** (removes a deprecated element — the only change that can break a
  consumer): requires sign-off from both Datolab and a `macrolibre.app`
  maintainer.

## Consequences

### Positive

- Consumers pin a semver tag and know, from the number alone, whether an
  upgrade is safe; a major is the only version that can require code changes
- Directly closes ADR-0004's contract-drift and governance risks — breaking
  changes are both rare and gated to the heavier approval
- Uses only standard OpenAPI 3.1 / HTTP mechanisms; no bespoke tooling
- Keeps additive evolution frictionless, which is the common case

### Negative / Trade-offs

- Requires ongoing discipline (add-alongside, deprecate, carry to a major)
  rather than editing the spec in place
- Deprecated elements linger in the contract through their sunset window,
  enlarging the surface temporarily
- A single repo-wide version means a change in one epic bumps the whole
  contract's version — simpler to pin, but noisier than per-document versions

### Risks

- Discipline can slip and a breaking change land in a minor by accident —
  mitigate with a CI diff-lint (e.g. `redocly` breaking-change detection)
  that fails when a minor/patch bump contains a breaking change
- The `0.x` phase permits breaking freely per semver, so the hard
  no-break guarantee only begins at `1.0.0`; consumers should treat any
  `0.x` pin as provisional

## Links

- Related: ADR-0004 (spec lives in a dedicated repo; left governance open —
  resolved here)
- Reference: Semantic Versioning 2.0.0 — https://semver.org
- Reference: RFC 8594 (the `Sunset` HTTP header)
- Distinct axis: dataset-release calendar versioning `vYYYY.MM` (SRS FR-C-1),
  see `openapi/releases.yaml`
