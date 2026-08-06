---
name: evolve-contract
description: >-
  Safely add to, change, deprecate, or remove anything in the OpenAPI contract
  under openapi/. Enforces ADR-0001: semver bumps, the deprecate-don't-break
  lifecycle, the oasdiff breaking-change gate, and change-type governance. Use
  whenever editing openapi/*.yaml — adding an endpoint or field, changing a
  shape, deprecating, or planning a version bump/tag.
---

# Evolve the contract

This repo is a public contract pinned by `datocal.com`, `macrolibre.app`, and
B2B integrators. A change that breaks a pinned consumer is a defect. Follow
this procedure; it implements [ADR-0001](../../../docs/adr/0001-version-the-contract-with-semver-and-a-deprecate-dont-break-policy.md)
and the project `CLAUDE.md`.

## 1. Classify the change

- **Additive** — new endpoint, new optional field, new enum-tolerant response.
  Safe → **minor**.
- **Fix / clarification** — description, example, non-normative wording.
  → **patch**.
- **Would-be-breaking** — removing or renaming a path/operation/parameter/field,
  making an optional field `required`, removing an enum value, tightening a
  type or `format`, changing status codes. **Do not apply as-is** → go to step 2.

## 2. Turn a breaking change into an additive one

Never edit the released element in place. Instead:

1. Add the new endpoint/field/schema **alongside** the existing one.
2. Mark the old one deprecated (valid on operations, parameters, and schemas):
   ```yaml
   deprecated: true
   x-deprecated-since: "<the version this ships in>"
   x-replaced-by: "<pointer to the replacement>"
   x-sunset: "<YYYY-MM-DD, when it may be removed>"
   ```
3. Note that the server SHOULD emit `Deprecation` + `Sunset` (RFC 8594)
   response headers for the deprecated element.

This keeps the change **minor**, not major. The old element is removed only in
a future **major**, on or after `x-sunset`.

## 3. Apply the edit

- Keep all four documents' `info.version` identical.
- Shared shapes (e.g. `Food`) live in `openapi/common/components.yaml` — edit
  there, not per-document.
- Use OpenAPI 3.1 nullables (`type: [x, 'null']`), never `nullable: true`.

## 4. Verify (both gates must pass)

```bash
npm run lint                              # zero warnings
oasdiff breaking <last-git-tag> openapi   # expect: no breaking changes
```

If `oasdiff` reports breaking changes and you intended a minor/patch, you
missed a deprecate-alongside step — return to step 2. A genuinely intended
breaking change means this is a **major** bump.

## 5. Choose the bump and update versions

Per the classification and the `oasdiff` result: major / minor / patch. Set the
new number in all four `info.version`. The git tag is cut separately (`vX.Y.Z`)
and is the version consumers pin — confirm with the user before tagging.

## 6. Governance gate

- **minor / patch** — proceed (Datolab approval).
- **major** (removes a deprecated element) — requires Datolab **and** a
  `macrolibre.app` maintainer sign-off. Do not cut a major tag without it.

## Never

- Edit or delete a released path/field/enum value in place.
- Let `info.version` drift between the four documents.
- Cut a tag, or push a breaking change, without explicit user go-ahead.
