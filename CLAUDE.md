# macrolibre-api-spec — project rules

This repo is a **public API contract** that `datocal.com` and `macrolibre.app`
pin a tagged version of (ADR-0004), and that B2B integrators build against.
Compatibility *is* the product. These rules implement
[ADR-0001](docs/adr/0001-version-the-contract-with-semver-and-a-deprecate-dont-break-policy.md);
that ADR is the source of truth if anything here is ambiguous.

When changing anything under `openapi/`, use the **`evolve-contract`** skill —
it walks the procedure below.

## Versioning

- **Semantic Versioning**, one repo-wide version — all four documents'
  `info.version` stay identical and match the git tag.
- **Do not hand-edit `info.version`.** The release workflow (ADR-0002,
  `.github/workflows/release.yml`) derives the bump from `oasdiff`, updates all
  four documents, writes `CHANGELOG.md`, and opens the release PR. Hand-editing
  breaks its tag-on-merge detection.
- The bump the workflow applies: **additive or deprecation → minor**;
  **fix/clarification → patch**; **removal or any breaking change → major**
  (phase-adjusted during `0.x`).
- After the release PR merges, a maintainer cuts the **GPG-signed** tag
  (`git tag -s vX.Y.Z`) — tags are never created in CI.
- Phases (ADR-0001): `0.0.x` alpha, `0.1.x`+ beta, `1.0.0` first stable.

## Compatibility — deprecate, don't break

Binding from `1.0.0`; applied as a goal during `0.x`.

- **Never edit or remove a released element in place** — path, operation,
  parameter, a `required` field, an enum value, or a type. That breaks pinned
  consumers.
- To change shape, **add the replacement alongside** the old one, then mark the
  old one:
  ```yaml
  deprecated: true
  x-deprecated-since: "1.3.0"
  x-replaced-by: "#/components/schemas/NewThing"
  x-sunset: "2027-01-01"
  ```
  (`deprecated: true` is valid on operations, parameters, and schemas.) The
  server SHOULD emit `Deprecation` and `Sunset` (RFC 8594) headers for these.
- **Remove a deprecated element only at the next major**, no earlier than its
  `x-sunset` date.

## Before every change

1. `npm run lint` — must be clean (zero warnings).
2. `oasdiff breaking <last-tag> <working>` — if it reports breaking changes,
   the bump MUST be major (or restructure as add-alongside + deprecate to keep
   it non-breaking).

## Governance (gated by change type)

- **minor / patch** (cannot break a consumer) — Datolab approval alone.
- **major** (removes a deprecated element — the only breaking kind) — requires
  Datolab **and** a `macrolibre.app` maintainer sign-off.

## Distinct version axes — do not conflate

- **This contract** is semver-versioned (git tag + `info.version`).
- **Dataset releases** (Epic C, `openapi/releases.yaml`) are calendar-versioned
  `vYYYY.MM` (FR-C-1). That versions data payloads, not the wire contract.
