# jwk_construction — JWK from an Ed25519 public key

> **Vectors:** *(implicit; tested via [`jwk_thumbprint`](jwk_thumbprint.md))*
> **Spec:** RFC 7517 / RFC 8037

## Algorithm

RFC 7517 / RFC 8037 OKP / Ed25519 JWK with three required fields:
`kty="OKP"`, `crv="Ed25519"`, `x=base64url(rawPublicKey)`.

## Java impl

- `Ed25519PublicKey.java:62-66` `jwk()`:
  `String x = Encoder.encodeToBase64(this.raw); return new Jwk("OKP", "Ed25519", x);`.
- `Jwk.java:13-27`: fields are declared as `kty`, `crv`, `x` with explicit
  `@JsonProperty`. Class is annotated `@JsonInclude(JsonInclude.Include.NON_NULL)`.
- The shared mapper sorts properties alphabetically
  (`Encoder.java:23` `MapperFeature.SORT_PROPERTIES_ALPHABETICALLY`),
  so JSON output is `{"crv":"Ed25519","kty":"OKP","x":"..."}`.

## TS impl

- `ts-definancy-sdk/src/auth/jwk.ts:17-23` `createJwk(publicKeyBytes)`
  returns `{ crv: "Ed25519", kty: "OKP", x: encodeBase64Url(publicKeyBytes) }`.
- The interface declares fields in alphabetical order
  (`jwk.ts:11-14`).

## Divergence notes

- **`x` encoding:** both produce base64url-no-padding (Java uses
  `Encoder.encodeToBase64`, urlSafe=true; TS uses `encodeBase64Url`).
  Byte-identical.
- **Field values:** identical literals.
- **Field order in JSON:** Java enforces it via Jackson's
  `SORT_PROPERTIES_ALPHABETICALLY`; TS *happens* to produce the right
  order via insertion order in the object literal. Both yield the same
  bytes, but TS depends on programmer discipline. A future contributor
  reordering keys in `createJwk` would silently break thumbprint parity
  with no test catching it unless conformance vectors exist.
