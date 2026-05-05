# Cross-language conformance

This directory holds the **language-agnostic correctness contract** for every
hand-written component of every SDK.

## The pattern

Crypto, identity, JWT signing, and amount math are reimplemented in each
language. Implementations diverge silently — and a divergence in DPoP
proof generation, JWK thumbprint, or ID checksum is a **security bug**,
not a feature gap.

The fix: **golden test vectors**. Each scenario file pins a fixed input
and the expected output computed against a trusted reference (RFC test
vector, well-known crypto library, hand-derivation from a published spec).
Vectors encode **correct** behavior, not bug-compatible behavior — see
`algorithms.md` for the divergences and known bugs they're designed to
catch.

## File layout

```
conformance/
├── README.md                  ← this file
├── algorithms/                ← per-domain reference docs (Java vs TS)
│   ├── README.md             ← index + cross-cutting divergence summary
│   └── <domain>.md           ← one doc per algorithm domain
└── vectors/
    └── <domain>/
        └── <scenario>.yaml   ← one scenario per file
```

The doc filenames in `algorithms/` mirror the directory names in `vectors/`
(e.g. `algorithms/base32.md` ↔ `vectors/base32/`) so you can find both
sides of a contract in one go.

One scenario file may contain multiple related **cases** (e.g. RFC 4648
publishes seven base32 known-answer pairs in one section — one file, seven
cases). Each file is self-contained and can be run in isolation.

## Vector schema

```yaml
name: "<domain>/<scenario>"          # mirrors path on disk
description: |
  <what this scenario asserts>
metadata:
  source: "<RFC section, paper, derivation>"
  spec:   "<the spec the algorithm conforms to>"
cases:
  - label:  "<short identifier for this case>"
    input:  { ... }                  # domain-specific, see encoding-suffix rules
    output: { ... }                  # domain-specific
  # ... more cases
```

For single-case scenarios, a one-element `cases:` list is still the shape —
keeps the runner code simple.

### YAML rules (mandatory, not stylistic)

YAML is forgiving in dangerous ways for security-critical vectors. The
following rules apply to every vector file:

1. **YAML 1.2 only.** Runners must load with a YAML 1.2 parser in safe
   mode (`js-yaml` ≥ v4 default; `SnakeYAML` with the v1.2 settings).
   YAML 1.1's `yes`/`no`/`on`/`off` boolean coercion is forbidden.
2. **Always quote string-valued fields.** Hex like `"66"`, base32 like
   `"MY"`, and any short token must be quoted. Unquoted `66` parses as
   integer; unquoted `MY` happens to be a string but only by accident.
   Quote everything that's a string.
3. **No anchors, tags, multi-document files, or custom types.** The
   subset is: scalars, sequences, mappings, comments. Effectively "JSON
   with comments and less quote noise."
4. **Comments are encouraged** for attribution, derivation notes, and
   explaining *why* an edge case matters. Keep them tight.
5. **Validate after load.** Runners must validate the parsed object
   against the schema before running cases. A vector with a missing
   `output` field should error loudly, not silently pass.

### Encoding suffix conventions

Field names carry their encoding to remove ambiguity:

| Suffix        | Meaning                                                       |
|---------------|---------------------------------------------------------------|
| `_hex`        | Lowercase hex of raw bytes (`"deadbeef"`)                     |
| `_b64url`     | Base64url, **no padding** (`-_` alphabet)                     |
| `_b64`        | Standard base64 (`+/` alphabet, padding present)              |
| `_utf8`       | A plain UTF-8 string (`"hello"`)                              |
| `_json`       | A pre-canonicalized JSON string (whitespace-significant)      |
| (no suffix)   | A plain string with no special encoding (e.g. an enum value)  |

Bytes that are conceptually opaque (a public key, a signature, a hash)
are always declared with their encoding suffix. Strings that are *intended*
to be exact bytes on the wire (canonical JSON, a signing input) use `_utf8`
or `_json` so the runner knows whether to compare them as code points or
as bytes.

## Active domains

| Domain                | Files                                       | Reference                  |
|-----------------------|---------------------------------------------|----------------------------|
| Base32 codec          | `base32/*.yaml`                             | RFC 4648 §6 + §10          |
| Base64url codec       | `base64url/*.yaml`                          | RFC 4648 §5                |
| SHA-256               | `sha256/*.yaml`                             | NIST FIPS 180-4            |
| SHA-512/256           | `sha512_256/*.yaml`                         | NIST FIPS 180-4 §6.7       |
| Ed25519               | `ed25519/*.yaml`                            | RFC 8032                   |
| DefinancyId checksum  | `id_checksum/*.yaml`                        | Definancy ID spec          |
| DefinancyDid parse    | `did_parse/*.yaml`                          | Definancy DID spec         |
| JWK thumbprint        | `jwk_thumbprint/*.yaml`                     | RFC 7638                   |
| JWT canonical         | `jwt_canonical/*.yaml`                      | RFC 7519 + Definancy spec  |
| Authorization JWT     | `authorization_jwt/*.yaml`                  | Definancy auth spec        |
| DPoP proof            | `dpop_proof/*.yaml`                         | RFC 9449 + Definancy spec  |
| DPoP body hash        | `dpop_body_hash/*.yaml`                     | Definancy auth spec        |
| Amount math           | `amount_math/*.yaml`                        | Definancy amount contract  |

## Runner contract

Each SDK's conformance runner must:

1. Discover all `*.yaml` files under `conformance/vectors/<domain>/` for
   the domains it implements.
2. Load each file with a YAML 1.2 safe loader; validate the parsed
   object against the schema before running cases.
3. For each case, decode `input` per the encoding-suffix conventions,
   execute the language's implementation, and compare against `output`
   byte-for-byte (after decoding `output` per the same conventions).
4. Emit one `PASS`/`FAIL` line per case in the form
   `<name>:<label> PASS|FAIL [diagnostic]`.
5. Exit non-zero if any case FAILs.

CI runs the runner for every SDK on every PR. Drift → red build.

## Authoring rules

- **Compute against a trusted reference, not against the SDK code.** Use
  RFC published test vectors, NIST KATs, or a third-party reference
  implementation. If no published vector exists for the exact algorithm
  (e.g. Definancy-specific composition like `id_checksum`), derive
  step-by-step from the spec and document the derivation in `description`.
- **Include intermediate values in `output` where helpful.** For
  composite algorithms (JWK thumbprint = canonical JSON → SHA-256 →
  base64url), include the intermediate `canonical_json_utf8` so a failing
  runner can isolate which step is wrong.
- **One scenario per file.** Don't bundle unrelated tests; that hides
  failures. Multiple `cases` are fine when they exercise the same
  algorithm with different inputs (RFC known-answer suites are the
  canonical example).
- **Stable inputs.** Use bytes/keys you can publish. Never embed real
  production keys.

> **Status:** Active. 13 scenarios committed across all 13 active domains;
> Java + TypeScript runners both green in CI as of factory-v0.0.1
> (2026-05-01). Edge-case coverage grows incrementally as gaps surface.
