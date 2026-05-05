# jwt_canonical — JWT canonical serialization

> **Vectors:** `vectors/jwt_canonical/` *(planned)*
> **Spec:** RFC 7519 + Definancy convention

## Algorithm

JWS Compact Serialization: `BASE64URL(UTF8(header_json)) || "." ||
BASE64URL(UTF8(claims_json)) || "." || BASE64URL(signature)`.

The bytes signed are the UTF-8 of `header_b64 || "." || claims_b64`
(the first two segments joined with a literal dot).

## Java impl

- `languages/java/sdk/src/main/java/com/definancy/sdk/auth/Header.java:31-38`,
  `auth/Claims.java:18-25`: each emits its JSON via the alphabetical-sort
  ObjectMapper, then UTF-8 bytes, then `Encoder.encodeToBase64`
  (urlSafe=true, no padding).
- `auth/Jwt.java:46-56` `encodeB64()`:
  `header.encodeB64() + "." + claims.encodeB64() (+ "." + signature)`.
- The signing input fed to `Signer.sign(...)` is `header_b64 + "." + claims_b64`
  (a Java String). `KeyPair.sign` UTF-8 encodes it before signing
  (`KeyPair.java:106`).

## TS impl

- `ts-definancy-sdk/src/auth/jwt.ts:29-38` `encode()`:
  ```ts
  const h = encodeBase64Url(new TextEncoder().encode(sortedJsonStringify(this.header)));
  const c = encodeBase64Url(new TextEncoder().encode(sortedJsonStringify(this.claims)));
  ```
- `jwt.ts:42-44` `sortedJsonStringify(obj)` calls `JSON.stringify(obj,
  Object.keys(obj).sort())`.
- The signing input is the same `h + "." + c` string.
  `KeyPair.sign` UTF-8 encodes via `TextEncoder`
  (`keypair.ts:124`).

## Divergence notes — **CONFIRMED BUG**

- **Top-level key sort:** Java sorts via Jackson; TS sorts via
  `Object.keys(obj).sort()`. Both produce ASCII/Unicode codepoint
  ordering. Identical for the field names used here.
- **Nested object key sort — CRITICAL:**
  - Jackson's `SORT_PROPERTIES_ALPHABETICALLY` is **recursive** by
    default — every POJO it serializes has its fields sorted.
  - TS's `sortedJsonStringify` uses `JSON.stringify(obj,
    Object.keys(obj).sort())` where the second arg is the **replacer
    array of allowed keys**, which `JSON.stringify` interprets as a
    *whitelist of top-level property names to include*. It does NOT
    recursively sort nested objects, and worse, **it filters nested
    objects by the same key list.** That is, `{cnf: {jkt: "..."}}` with
    the replacer `["aud","cnf","exp","iat","iss","sub"]` would have the
    `cnf` value's `jkt` field filtered out **if `jkt` is not in the
    replacer list**.
  - Per MDN: when a replacer array is provided, `JSON.stringify` includes
    only properties whose key is in the array, **at every level**. So
    `{"cnf":{"jkt":"..."}}` becomes `{"cnf":{}}` in TS today, while Java
    emits the full `{"cnf":{"jkt":"..."}}`.
  - **Verified via Node REPL:**
    `JSON.stringify({a:{b:1}}, ["a"])` → `'{"a":{}}'`.
  - **This is a confirmed serialization bug in the TS implementation.**
    The Authorization JWT's `cnf.jkt` is silently stripped before
    signing/encoding, and the DPoP header's embedded `jwk` becomes `{}`.
    Every TS-issued JWT today is malformed against the spec and against
    Java's output.
- **Whitespace:** both emit JSON with no whitespace.
- **Number formatting:** `iat`/`exp` are integer Unix seconds. Java
  `Long` serializes as decimal; JS `JSON.stringify` of an integer Number
  serializes as decimal. Identical.
- **String escaping:** all values here are ASCII (DIDs, base64url
  thumbprints, audience URLs); neither side needs to escape, so
  outputs match.
- **Base64url segment encoding:** see [`base64url.md`](base64url.md) —
  identical byte output.
- **Signature input bytes:** both languages take the concatenation
  `h + "." + c`, UTF-8 encode, then sign. Bytes signed are identical
  *iff* the JSON segments are identical (see "Nested object key sort"
  above).
