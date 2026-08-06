# macrolibre-api-spec

The public OpenAPI 3.1 contract between the Kcal backend
([`datocal.com`](https://git.datolab.com/sites/datocal.com), proprietary) and the
Kcal client ([`macrolibre.app`](https://github.com/Datolab/macrolibre.app), AGPL).

This repo is the single source of truth for the wire contract. It lives on its
own, separate from either implementation, per ADR-0004: both sides pin a specific
tagged version of this spec and generate/verify against it.

## Layout

One OpenAPI document per SRS epic, sharing common building blocks via relative
`$ref` into `openapi/common/components.yaml`:

| Document | SRS epic | Scope |
|---|---|---|
| `openapi/foods.yaml` | Epic B (FR-B-1..7) | Food search, verified lookups, community submissions |
| `openapi/releases.yaml` | Epic C (FR-C-1..5) | Versioned, signed dataset releases and delta-sync |
| `openapi/sync.yaml` | Epic E (FR-E-2..5) | End-to-end-encrypted blob storage for Pro multi-device sync |
| `openapi/billing.yaml` | Epic F (FR-F-3) | Client-facing checkout entry point |

`openapi/common/components.yaml` holds the shared security schemes (three auth
tiers), cursor-pagination parameters, rate-limit headers, and the
`TooManyRequests` response — implementing the throttling header contract from
ADR-0005.

## Working with the spec

```bash
npm install
npm run lint     # validate all four documents with redocly
npm run bundle   # emit self-contained documents to dist/
```

## Status

**Alpha (`0.0.1`).** All four epics (B/C/E/F) have designed, lint-clean
endpoints and schemas. Versioning and compatibility policy is proposed in
[ADR-0001](docs/adr/0001-version-the-contract-with-semver-and-a-deprecate-dont-break-policy.md).
Not yet tagged or published to a remote.

## License

Released into the public domain under [CC0 1.0 Universal](./LICENSE).
