# authorization_jwt

> **Vectors:** `vectors/authorization_jwt/` *(planned)*
> **Spec:** Definancy auth spec (DPoP-bound JWS-EdDSA)

## Algorithm

JWS-EdDSA token identifying the caller via DID and binding to the DPoP key.

- Header: `{ "alg": "EdDSA", "typ": "JWT" }`.
- Claims:
  - `iss`: caller's DID string.
  - `sub`: caller's DID string (same as `iss`).
  - `aud`: scheme + host (and explicit port, if any) of the request URL.
  - `iat`: now in **seconds** since Unix epoch.
  - `exp`: `iat + 60`.
  - `cnf`: `{ "jkt": jwkThumbprint(jwk) }`.

## Java impl

- `languages/java/sdk/src/main/java/com/definancy/sdk/auth/AuthorizationHeader.java:7-9`:
  `super("JWT")`. Header has `alg="EdDSA"` (final) and `typ="JWT"`.
- `auth/AuthorizationClaims.java:8-25`: declares fields with
  `@JsonProperty`; nested `ConfirmationClaims` has `@JsonProperty("jkt")`.
  `@JsonInclude(JsonInclude.Include.NON_NULL)` at the class level.
- `auth/Authorization.java:8-24`: constructor sets `iss=sub=did.toString()`,
  `aud=audience`, `iat=Instant.now().getEpochSecond()`, `exp=iat+60`,
  `confirmation.thumbprint=thumbprint`.
- `auth/impl/LocalAuthProvider.java:23-39`: builds audience as
  `URIBuilder.setScheme(...).setHost(...).setPort(uri.getPort()).build()`.
  When `uri.getPort()` is `-1` (no explicit port), the result has no
  port. When the port is explicit (even if it's the default 443/80),
  it is preserved in the audience string. Calls `signer.sign(authorization.encodeB64())`.

## TS impl

- `ts-definancy-sdk/src/auth/authorization.ts:14-38` `createAuthorizationJwt`:
  builds the audience-less JWT (audience passed in). Header is
  `{alg, typ}`; claims are `{aud, cnf:{jkt}, exp, iat, iss, sub}` —
  literal in alphabetical order.
- `ts-definancy-sdk/src/auth/local-provider.ts:30-54` `authenticate`:
  builds audience as `` `${parsed.protocol}//${parsed.host}` ``.
  WHATWG URL `host` strips default ports (so `https://api:443/foo`
  becomes `https://api`). Calls `signer.sign(authJwt.encode())`.

## Divergence notes

- **Audience formatting (`LocalAuthProvider`):**
  - Java preserves explicit port even when default
    (`https://api.example.com:443`).
  - TS strips default port via WHATWG URL `host`.
  - **Same input URL `https://api.example.com:443/foo` produces
    different `aud` claims in the two SDKs.** This will produce
    non-byte-equal JWTs and non-equal signatures. **Treat as a divergence
    bug.** Test vectors must avoid explicit default ports if they expect
    parity — or pin the audience explicitly via mocking.
- **`iat`/`exp` granularity:** seconds in both. Both call "now"; for
  conformance, vectors must inject a clock or compare modulo timestamp.
- **`cnf` field-stripping bug** (see [`jwt_canonical.md`](jwt_canonical.md)):
  TS's `sortedJsonStringify` filters nested object keys, so `cnf.jkt` is
  emitted as `{}`, breaking RFC 7638-bound DPoP. **Confirmed via Node
  REPL.** Until fixed, the TS Authorization JWT is unusable.
- **Signing flow:** otherwise identical.
