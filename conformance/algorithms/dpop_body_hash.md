# dpop_body_hash — body hash for the DPoP `bsh` claim

> **Vectors:** `vectors/dpop_body_hash/` *(planned)*
> **Spec:** Definancy auth spec (custom `bsh` claim)

## Algorithm

`bsh = base64url( SHA-512/256( body_bytes ) )`, included only when a body
is present.

## Java impl

- `languages/java/sdk/src/main/java/com/definancy/sdk/auth/DPoP.java:13-17`:
  ```java
  if (body != null) {
      byte[] bodyBytes = body.getBytes(StandardCharsets.UTF_8);
      byte[] bodyDigest = Digester.digest(bodyBytes);
      bodyHash = Encoder.encodeToBase64(bodyDigest);
  }
  ```
- The string passed in is `body.toString()` of the JAX-RS entity object
  (`LocalAuthProvider.java:49`), **not the wire JSON.**

## TS impl

- `ts-definancy-sdk/src/auth/dpop-proof.ts:26-30`:
  ```ts
  let bodyHash: string | undefined;
  if (body !== null && body.length > 0) {
      const digest = await sha512_256(body);
      bodyHash = encodeBase64Url(digest);
  }
  ```
- The bytes passed in come from `request.clone().arrayBuffer()`
  (`ts-definancy-sdk/src/auth/dpop.ts:30-32`) — the actual wire bytes.

## Divergence notes

- **Empty-body semantics:**
  - Java: `body == null` → no `bsh`. `body == ""` → hashes empty string,
    `bsh` present. (`null` is what `requestContext.getEntity()` returns
    when there is no entity; an empty-string entity would hash to the
    32-byte digest of `""`.)
  - TS: `body == null` OR `body.length == 0` → no `bsh`. **No way to
    produce a `bsh` of an empty body.**
  - Same wire request (an empty POST) produces different DPoP claims.
- **What is hashed — CONFIRMED BUG:**
  - Java: UTF-8 bytes of `entity.toString()`. Unless every entity POJO
    overrides `toString()` to emit JSON, this is **`Class@hex` JVM
    identity garbage**, not the wire body. **Confirmed by code reading.**
  - TS: UTF-8 bytes of the actual serialized wire body.
  - **Even if both sides agreed on the "empty body" semantics, the
    contents being hashed are not the same data.** Highest-priority
    finding in this set.
- **Encoding of the digest:** both base64url-no-padding via the same
  primitives. Same encoding once the input is the same.
- **Algorithm name:** the SDK calls this `bsh`, not RFC 9449's `ath`.
  Vectors must use `bsh`.
