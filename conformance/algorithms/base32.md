# base32

> **Vectors:** [`vectors/base32/`](../vectors/base32/)
> **Spec:** RFC 4648 §6

## Algorithm

RFC 4648 Base32 alphabet (`A-Z2-7`), **no padding** in encoded output, used
exclusively to encode the 36-byte `id||checksum` blob into a 58-character
human ID string.

## Inputs / outputs

- Encode: `Uint8Array` / `byte[]` → `String` (base32, no `=` padding).
- Decode: `String` (with or without `=` padding) → bytes.

## Java impl

- `languages/java/sdk/src/main/java/com/definancy/sdk/util/Encoder.java:54-58`
  `encodeToBase32(bytes)`: builds an Apache Commons `Base32` with pad char `=`,
  encodes, then strips trailing `=` via `StringUtils.stripEnd`. Apache Commons
  uses the standard RFC 4648 alphabet `ABCDEFGHIJKLMNOPQRSTUVWXYZ234567`.
- `Encoder.java:65-68` `decodeFromBase32(str)`: identical codec, decode allows
  padding to be present or missing.

## TS impl

- `ts-definancy-sdk/src/crypto/encoder.ts:28` constant
  `BASE32_ALPHABET = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567"`.
- `encoder.ts:30-49` `encodeBase32(bytes)`: bit-shift accumulator, emits 5-bit
  symbols. Final partial group is left-shifted to fill the missing bits with
  zeros (`(value << (5 - bits)) & 0x1f`). No `=` padding.
- `encoder.ts:51-78` `decodeBase32(str)`: strips trailing `=`, builds a
  lookup map, **uppercases each input character** (`char.toUpperCase()`),
  bit-accumulates back to bytes. Throws on unknown characters.

## Divergence notes

- **Alphabet:** identical (`A-Z2-7`).
- **Padding on encode:** identical (both strip).
- **Case sensitivity on decode:** TS accepts lowercase via
  `char.toUpperCase()` (`encoder.ts:65`). Apache Commons `Base32` accepts
  lowercase by default. Should match.
- **Stray characters on decode:** TS throws `Invalid base32 character` on any
  non-alphabet character (`encoder.ts:67`). Apache Commons `Base32` defaults
  to lenient behavior — by default it silently skips characters it does not
  recognize (whitespace, etc). For 58-char Definancy IDs this is unreachable
  in normal use, but a malformed input could behave differently across SDKs.
  Conformance vectors that probe error paths must keep this in mind.
- **Final-group padding bits:** the TS encoder does not validate that decoder
  inputs have zeroed pad bits. Apache Commons rejects non-zero pad bits in
  some configurations. Edge case for malformed inputs only.
