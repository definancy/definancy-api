# base64url

> **Vectors:** `vectors/base64url/` *(planned)*
> **Spec:** RFC 4648 §5

## Algorithm

RFC 4648 §5 base64url, **no padding** on encode, used everywhere a binary
blob needs to appear inside a JWT segment, JWK `x` value, JWK thumbprint,
or signature.

## Inputs / outputs

- Encode: bytes → `String` over alphabet `[A-Za-z0-9-_]`, no `=`.
- Decode: `String` (with or without padding, `-_` or `+/`) → bytes.

## Java impl

- `Encoder.java:75-78` `encodeToBase64(bytes)`: constructs Apache Commons
  `Base64(0, null, true)` — line length 0, no separator, **urlSafe=true**.
  Per Apache Commons Codec 1.19 docs (used here, see `pom.xml`), urlSafe
  output uses the `-_` alphabet AND **omits `=` padding entirely**.
- `Encoder.java:85-88` `decodeFromBase64(str)`: same codec, decoder accepts
  both alphabets and tolerates missing padding.

## TS impl

- `ts-definancy-sdk/src/crypto/encoder.ts:5-11` `encodeBase64Url(bytes)`:
  builds a binary string char-by-char, calls `btoa(...)`, then runs
  `.replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "")`.
- `encoder.ts:13-22` `decodeBase64Url(str)`: replaces `-` with `+`, `_` with
  `/`, and calls `atob`. **The variable is named `padded` but no padding
  characters are added.**

## Divergence notes

- **Encode output:** byte-identical (URL-safe alphabet, no padding).
- **Decode of strings missing `=` padding:** Java is lenient. TS calls
  `atob(padded)` — Node's `atob` accepts unpadded input but the WHATWG/HTML
  spec requires `atob` to throw on inputs whose length is not a multiple of
  4. Browsers historically throw; Node currently does not. So
  `decodeBase64Url("abc")` works in Node but may throw in a browser. This
  is a runtime-environment divergence inside the TS implementation, not a
  Java-vs-TS divergence per se.
- **Loose-padding round-trip:** `decodeFromBase64(encodeToBase64(x))` ==
  `x` in both languages. Standard-alphabet (`+/`) inputs round-trip too.
- **Ed25519 secret import** (`KeyPair.fromSecret` / `generateKeyPairFromSecret`):
  Java decodes via `Encoder.decodeFromBase64`, which accepts both standard
  base64 and base64url, with or without padding. TS's `decodeBase64Url`
  tolerates only base64url (`-_`) and inherits `atob`'s padding behavior.
  An exported secret is produced by both as base64url-no-padding
  (`KeyPair.export()`), so happy-path round-trip matches; if a caller hands
  in a standard-alphabet (`+/`) secret, Java accepts it and TS may not.
