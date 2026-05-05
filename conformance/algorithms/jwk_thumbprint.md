# jwk_thumbprint — RFC 7638

> **Vectors:** [`vectors/jwk_thumbprint/`](../vectors/jwk_thumbprint/)
> **Spec:** RFC 7638

## Algorithm

RFC 7638 §3 for an OKP/Ed25519 key:

1. Build the canonical JSON containing only the required fields, with
   keys ordered lexicographically: `{"crv":"Ed25519","kty":"OKP","x":"..."}`.
2. UTF-8 encode that string with no whitespace.
3. SHA-256.
4. Base64url-encode the digest, no padding.

## Java impl

- `Jwk.java:30-32` `json()` → uses the alphabetical-sort ObjectMapper.
- `Jwk.java:35-37` `encode()` → UTF-8 bytes of `json()`.
- `Jwk.java:39-43` `thumbprint()`: SHA-256 the encoded bytes via
  `DigestUtils.sha256`, then `Encoder.encodeToBase64` (urlSafe=true,
  no padding).

## TS impl

- `ts-definancy-sdk/src/auth/jwk.ts:32-37` `jwkThumbprint(jwk)`:
  ```ts
  const canonical = JSON.stringify({ crv: jwk.crv, kty: jwk.kty, x: jwk.x });
  const hash = await sha256(new TextEncoder().encode(canonical));
  return encodeBase64Url(hash);
  ```

## Divergence notes

- **Canonical string identity:**
  - Java emits `{"crv":"Ed25519","kty":"OKP","x":"..."}`. Jackson default
    serialization includes no whitespace; alphabetical sort is enforced
    globally.
  - TS calls `JSON.stringify(...)` with no replacer; ECMAScript guarantees
    no whitespace and uses object property insertion order. The literal
    `{ crv, kty, x }` is constructed in alphabetical order, so output is
    the same string.
  - **Both produce byte-identical canonical JSON for OKP/Ed25519 JWKs.**
- **String escaping inside `x`:** the `x` value is base64url, which uses
  only ASCII characters that JSON does not need to escape. No divergence.
- **Hash & base64url encoding:** byte-identical (see
  [`base64url.md`](base64url.md) and [`sha256.md`](sha256.md)).
- **Robustness concern:** The TS implementation does NOT use the
  `sortedJsonStringify` helper from `jwt.ts`; it relies on the literal's
  insertion order. If `Jwk` ever gains another field (e.g., `kid`), TS
  would include or exclude it based on the literal in `jwkThumbprint`,
  while Java would include/exclude based on Jackson's `NON_NULL` filter
  on the class. **Adding any new JWK field is a high-risk change for
  cross-language thumbprint parity.**
