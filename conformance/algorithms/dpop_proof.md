# dpop_proof

> **Vectors:** `vectors/dpop_proof/` *(planned)*
> **Spec:** RFC 9449 (customized) + Definancy auth spec

## Algorithm

Per RFC 9449, customized:

- Header: `{ "alg": "EdDSA", "jwk": {crv,kty,x}, "typ": "DPoP+JWT" }`.
- Claims:
  - `jti`: random UUID v4 string.
  - `htm`: HTTP method.
  - `htu`: request URI without query and fragment (per RFC 9449 §4.2 —
    scheme + authority + path).
  - `iat`: now (seconds).
  - `exp`: `iat + 60`.
  - `bsh`: SHA-512/256 of body bytes, base64url. **Custom name** —
    RFC 9449 specifies `ath` for an *access-token* hash, not a body hash.
    This SDK invents `bsh` to mean "body sha".

## Java impl

- `languages/java/sdk/src/main/java/com/definancy/sdk/auth/DPoPHeader.java:7-18`:
  extends `Header`, adds `jwk` field. Constructor calls `super("DPoP+JWT")`.
- `auth/DPoPClaims.java:8-20`: fields `jti, htm, htu, iat, exp, bsh`. With
  Jackson's alphabetical sort: serialized order is `bsh, exp, htm, htu, iat, jti`.
- `auth/DPoP.java:9-33`: constructor takes
  `(id, method, uri, body, jwk)`; if `body != null`, computes
  `Digester.digest(body.getBytes(UTF8))` (= SHA-512/256) and
  `Encoder.encodeToBase64(...)` (= base64url-no-padding) into `bsh`.
- `auth/impl/LocalAuthProvider.java`:
  - `id = UUID.randomUUID().toString()`.
  - `body = requestContext.getEntity()` — the deserialized request body
    object. For non-String entities, serialized via the SDK's shared
    Jackson mapper (`Encoder.encodeToJson`) so the hashed bytes match
    the wire bytes Jersey emits. For String entities, the bytes are
    taken directly.
  - `htu` is built from the request URI's scheme + host + port + path
    (no query, no fragment) — RFC 9449 §4.2 compliant.
- Resulting header JSON (alpha-sorted): `{"alg":"EdDSA","jwk":{"crv":"Ed25519","kty":"OKP","x":"..."},"typ":"DPoP+JWT"}`.

## TS impl

- `ts-definancy-sdk/src/auth/dpop-proof.ts:17-51` `createDpopProof`:
  - If `body !== null && body.length > 0`, computes
    `sha512_256(body)` and `encodeBase64Url(digest)` → `bsh`.
  - If `body === null` OR `body.length === 0`, **`bsh` is omitted from
    claims entirely.**
  - Header: `{alg:"EdDSA", jwk:{crv,kty,x}, typ:"DPoP+JWT"}` (literal
    in alphabetical order).
  - Claims: `{exp, htm, htu, iat, jti}` (no `bsh` until added; literal
    in alphabetical order).
- `ts-definancy-sdk/src/auth/local-provider.ts`:
  - `jti = crypto.randomUUID()` (UUID v4).
  - `body` is the actual UTF-8 bytes of the wire body
    (`request.clone().arrayBuffer()` in `dpop.ts:30-32`).
  - `htu` is built as `${parsed.protocol}//${parsed.host}${parsed.pathname}`
    — RFC 9449 §4.2 compliant, matching Java.

## Divergence notes (highest-risk domain)

- **Body bytes hashed are different things — CONFIRMED BUG:**
  - Java hashes **`Object.toString()` of the JAX-RS entity**
    (`LocalAuthProvider.java:49`). Unless every request body type
    overrides `toString()` to emit the wire JSON (which is not
    standard and not done here), this hashes a JVM identity string —
    `Class@hashcode` — completely uncorrelated with the wire bytes.
  - TS hashes **the actual wire body bytes** (`dpop.ts:29-32`).
  - **The two SDKs hash entirely different inputs for the same logical
    request.** A server validating `bsh` against the wire body would
    accept TS DPoP proofs and reject Java DPoP proofs (or vice versa,
    depending on what it hashes). **This is an active correctness
    bug.** That this hasn't blown up in practice means the server
    probably isn't validating body hash today.
- **Empty body handling:**
  - Java: any non-null body, including `body.toString()` of `""`
    (which would be `""`), is hashed. `""` becomes a 32-byte digest.
    The `bsh` claim is included.
  - TS: a zero-length body is treated as no body — `bsh` is omitted
    entirely.
  - **Same logical "empty POST body" produces a `bsh` claim in Java and
    no `bsh` claim in TS.** Different JSON, different signed bytes.
- **`htu` value:** both set this to the request URI without query and
  fragment (scheme + authority + path), matching RFC 9449 §4.2. The
  audience-formatting divergence from
  [`authorization_jwt.md`](authorization_jwt.md) still applies (Java keeps
  explicit default ports, TS strips them) and now affects `htu` too,
  since the host[:port] portion comes from the same source.
- **`jti` randomness:** UUID v4 in both languages, by spec
  indistinguishable in format. Test vectors cannot pin it without
  injecting deterministic randomness on both sides.
- **JWK in header:** JSON sub-object `{crv, kty, x}` — Jackson sorts
  alphabetically; TS object literal is in alphabetical order. Both
  produce `{"crv":"...","kty":"...","x":"..."}`. Same bytes.
- **Header `typ`:** both `"DPoP+JWT"`. Same.
- **Sort-order trap (see [`jwt_canonical.md`](jwt_canonical.md)):** the
  same `sortedJsonStringify` filtering bug applies here. The DPoP header
  has nested `jwk` whose keys are `crv, kty, x`; the top-level replacer
  array is `["alg", "jwk", "typ"]`. Per the JSON.stringify replacer-array
  contract, the nested `jwk` object's keys (`crv, kty, x`) are not in
  the array and would be filtered. **Confirmed via Node REPL — TS DPoP
  headers are emitted as `{"alg":"EdDSA","jwk":{},"typ":"DPoP+JWT"}` —
  the JWK is empty.** Same root cause as the `cnf.jkt` issue.
