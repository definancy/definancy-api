# id_checksum — DefinancyId (32 bytes + 4-byte checksum, Base32 encoded)

> **Vectors:** `vectors/id_checksum/` *(planned)*
> **Spec:** Definancy ID spec

## Algorithm

1. Take the 32-byte raw Ed25519 public key as the ID material.
2. Compute SHA-512/256 over those 32 bytes.
3. Take the **last 4 bytes** of the digest as the checksum.
4. Concatenate: `id (32) || checksum (4)` = 36 bytes.
5. Base32-encode (no padding) → 58 characters (`ceil(36*8/5) = 58`).

## Inputs / outputs

- Construct from a 32-byte public key, raw 32-byte ID, or a 58-char string.
- String form: 58 chars over `[A-Z2-7]`.

## Java impl

- `languages/java/sdk/src/main/java/com/definancy/sdk/ID.java:15-16` constants:
  `LEN_BYTES = 32`, `CHECKSUM_LEN_BYTES = 4`, `EXPECTED_STR_ENCODED_LEN = 58`.
- `ID.java:25-28` constructor from `Ed25519PublicKey`: copies 32 bytes.
- `ID.java:42-66` constructor from `String`: validates length 58, base32-decodes,
  splits the last 4 bytes as supplied checksum, recomputes
  `Digester.digest(idBytes)` (= SHA-512/256), takes the last 4 bytes
  (`Arrays.copyOfRange(hashedId, LEN_BYTES - CHECKSUM_LEN_BYTES, hashedId.length)` —
  i.e. bytes 28..32 of a 32-byte digest = the last 4 bytes), compares with
  `Arrays.equals` (NOT constant-time).
- `ID.java:76-92` `encodeAsString()`: same hash → last-4 → concat → base32.

## TS impl

- `ts-definancy-sdk/src/identity/id.ts:4-6` constants identical.
- `id.ts:38-43` `fromPublicKey`: validates 32 bytes, stores.
- `id.ts:54-79` `fromString`: validates length 58, base32-decode, splits last 4
  bytes, recomputes `sha512_256(idBytes).slice(LEN_BYTES - CHECKSUM_LEN_BYTES)`,
  compares with **constant-time** `constantTimeEqual` (`id.ts:113-120`).
- `id.ts:95-109` `toString()`: hash, last-4 slice, concat, base32-encode,
  asserts length is 58.

## Divergence notes

- **Algorithm:** identical.
- **Checksum extraction expression:**
  - Java: `Arrays.copyOfRange(hashedId, LEN_BYTES - CHECKSUM_LEN_BYTES,
    hashedId.length)`.
  - TS: `hash.slice(LEN_BYTES - CHECKSUM_LEN_BYTES)`.
  - Both pull bytes 28..32 of the 32-byte SHA-512/256 digest. Equivalent.
- **Constant-time compare:** TS uses constant-time; Java does not. No byte-level
  divergence, but a low-grade timing oracle in Java when validating untrusted
  IDs.
- **Real divergence inherited from [`sha512_256.md`](sha512_256.md):**
  because TS's SHA-512/256 will fail at runtime, `toString()`/`fromString()`
  cannot succeed on WebCrypto-only platforms today. The encoding logic
  itself, once a working SHA-512/256 is in place, is byte-identical.
