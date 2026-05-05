# Algorithm reference: Java vs TypeScript SDKs

Per-domain reference for every hand-written cryptographic or identity
routine in the two SDK implementations. Each file documents what the
code *actually does*, including the legacy bugs the conformance vectors
were designed to catch.

> **Snapshot vs current state.** The per-domain files (`base32.md`,
> `jwt_canonical.md`, `dpop_body_hash.md`, …) are a faithful snapshot
> of the legacy code as analyzed on 2026-04-28. They preserve the
> "CONFIRMED BUG" annotations because the bugs were real at that time
> and the file:line citations match the legacy source.
>
> **All three confirmed bugs have since been fixed.** See the
> "Cross-cutting summary" section below for the resolution map. The
> per-domain files are kept as-is so the historical reasoning behind
> each fix remains traceable from the same place future divergences
> would be analyzed.

Source files inspected at snapshot time:

- Java: `languages/java/sdk/src/main/java/com/definancy/sdk/`
- TypeScript: `ts-definancy-sdk/src/` (legacy path; now
  `languages/typescript/sdk/src/`)

All `file_path:line` references are relative to the repository root.

## Index

| Domain                                     | Vectors                                            | Status           |
|--------------------------------------------|----------------------------------------------------|------------------|
| [base32](base32.md)                        | [`vectors/base32/`](../vectors/base32/)            | active           |
| [base64url](base64url.md)                  | [`vectors/base64url/`](../vectors/base64url/)      | active           |
| [sha256](sha256.md)                        | [`vectors/sha256/`](../vectors/sha256/)            | active           |
| [sha512_256](sha512_256.md)                | [`vectors/sha512_256/`](../vectors/sha512_256/)    | active           |
| [ed25519](ed25519.md)                      | [`vectors/ed25519/`](../vectors/ed25519/)          | active           |
| [id_checksum](id_checksum.md)              | [`vectors/id_checksum/`](../vectors/id_checksum/)  | active           |
| [did_parse](did_parse.md)                  | [`vectors/did_parse/`](../vectors/did_parse/)      | active           |
| [jwk_construction](jwk_construction.md)    | *(implicit; tested via jwk_thumbprint)*            | n/a              |
| [jwk_thumbprint](jwk_thumbprint.md)        | [`vectors/jwk_thumbprint/`](../vectors/jwk_thumbprint/) | active     |
| [jwt_canonical](jwt_canonical.md)          | [`vectors/jwt_canonical/`](../vectors/jwt_canonical/)         | active     |
| [authorization_jwt](authorization_jwt.md)  | [`vectors/authorization_jwt/`](../vectors/authorization_jwt/) | active     |
| [dpop_proof](dpop_proof.md)                | [`vectors/dpop_proof/`](../vectors/dpop_proof/)               | active     |
| [dpop_body_hash](dpop_body_hash.md)        | [`vectors/dpop_body_hash/`](../vectors/dpop_body_hash/)       | active     |
| [amount_math](#)                           | [`vectors/amount_math/`](../vectors/amount_math/)             | active     |

## Cross-cutting summary of divergences

The five divergences flagged at snapshot time, with their resolution.

### Confirmed bugs — all fixed

| # | Bug | Location | Resolution |
|---|-----|----------|-----------|
| 1 | DPoP body hash hashes wrong bytes (`entity.toString()`) | `LocalAuthProvider.java:49` | **Fixed** in factory commit `46d133f` (Java SDK `68722e6`) — DPoP API takes `byte[]`; `LocalAuthProvider` serializes the entity to wire bytes via the shared Jackson mapper before passing in. |
| 2 | `sortedJsonStringify` filters nested keys (recursive whitelist) | `jwt.ts:43` | **Fixed** in factory commit `abe220b` (TS SDK `4618fee`) — `JSON.stringify` replaced with a `canonicalize` helper that pre-rebuilds every object with sorted keys, then `JSON.stringify` with no replacer. |
| 3 | `sha512_256` calls non-existent WebCrypto algorithm | `digester.ts:6` | **Fixed** in factory commit `7bdcec2` (TS SDK `3c779ca`) — replaced `crypto.subtle.digest("SHA-512/256", …)` with `@noble/hashes/sha2.sha512_256` (audited pure-JS, portable across Node and browser). |

Each fix landed alongside the conformance vector that catches it
(`dpop_body_hash`, `jwt_canonical`, and `sha512_256` respectively),
demonstrating a clean red→green transition.

### Non-bug divergences — also closed

| # | Divergence | Location | Resolution |
|---|------------|----------|-----------|
| 4 | Empty-body DPoP: Java emitted `bsh` for `body!=null`; TS only for `length>0` | `DPoP.java`, `dpop-proof.ts` | **Aligned** in factory commit `46d133f` — Java now uses `body!=null && length>0`, matching TS and the practical "no body to hash" interpretation in HTTP. |
| 5 | Audience formatting on explicit default port (Java kept `:443`, TS stripped via WHATWG `URL.host`) | `LocalAuthProvider.java`, `local-provider.ts` | **Open** at the source-code level (the legacy port behavior remains), but the conformance vectors avoid explicit default ports so the divergence isn't currently exercised. Worth a follow-up if any production caller hits `https://api:443/…`. |

## Items that still need human attention

- **`KeyPair.java:128-156` `FixedSecureRandom`** — index-advance bug is
  harmless under current BC call patterns but worth pinning with a test
  that calls `nextBytes` twice with mismatched lengths.
- **Audience default-port handling** (divergence #5 above) — not
  exercised by current vectors. If a production caller hits
  `https://api:443/…`, Java and TS would diverge silently. Worth either
  fixing the URL formatting on one side or adding an explicit-port
  conformance vector.
- **Server-side body-hash validation** — does the server today actually
  verify `bsh` against the wire body, or accept anything? Now that the
  client side is fixed, knowing the server's behavior tells us whether
  any production tokens were silently accepted with the old buggy
  `bsh` values.
