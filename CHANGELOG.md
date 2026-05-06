# Changelog

All notable changes to the Definancy spec will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
Pre-1.0 releases use MINOR for any contract-visible change (new paths,
vector-value updates, layout changes); PATCH is reserved for documentation-
only fixes.

## [Unreleased]

## [0.4.0] - 2026-05-06

### Added
- **`GET /v1/experimental/ping`** — connectivity probe for the
  Experimental tag surface. The endpoint is unauthenticated and returns
  the standard `Status` schema. Implementation is intentionally minimal;
  its primary purpose is to keep the `Experimental` tag carrying at least
  one operation so that downstream code generators (notably
  openapi-generator-cli for Java, which skips writing API classes for
  empty tags) always regenerate the corresponding API class against the
  current spec — preventing stale methods from persisting after operations
  move out of the tag.

## [0.3.1] - 2026-05-05

### Changed
- Repo-level `README.md` and `api/README.md` rewritten for a public
  audience: removed references to private downstream consumers, removed
  cross-references into the (private) consumer factory's tooling paths,
  and gave this repo its own security-reporting contact instead of
  redirecting to a private repo's `SECURITY.md`. Documentation only —
  no spec or vector content changed.

## [0.3.0] - 2026-05-05

### Changed
- **Layout rename:** `openapi/oapi.yaml` → `api/openapi.yaml` (directory
  and filename both adopt more conventional names). Consumers must
  update their submodule pointer and any path references.

### Added
- `README.md` (top-level repo overview).
- `api/README.md` (OpenAPI spec render/lint specifics).
- `CHANGELOG.md` — this file. Backfilled for 0.1.0 + 0.2.0.

## [0.2.0] - 2026-05-05

### Added
- **Conformance suite consolidated into the spec repo:**
  `conformance/algorithms/*.md` (15 markdown algorithm specs covering
  base32/base64url, SHA-256, SHA-512/256, Ed25519, JWK construction +
  thumbprint, identity checksum + DID parsing, canonical JWT, Authorization
  JWT, DPoP proof + body hash, amount math) and `conformance/vectors/*.yaml`
  (13 vector files, 77 cases). Sourced from the SDK factory (`definancy-sdks`)
  where they previously lived. Allows every spec consumer to pull a single
  submodule for the full contract.

### Fixed
- **DPoP `htu` claim** corrected to RFC 9449 §4.2 (scheme + authority +
  path; no query, no fragment), replacing the previous audience-only value.
  Affects `dpop_proof/local_provider_at_t1700000000` and
  `dpop_body_hash/local_provider_with_body` vectors.

### Changed
- Repository renamed `definancy-api` → `definancy-spec` on GitHub. The
  prior URL still redirects, but consumers should update `.gitmodules`.

## [0.1.0] - 2026-04-30

### Added
- Initial Definancy OpenAPI spec — OAS 3.1, 20 component descriptions,
  Spectral house rules clean (`info.contact`, Velocity tag, vault-scope
  op descriptions).
- `LICENSE`.
