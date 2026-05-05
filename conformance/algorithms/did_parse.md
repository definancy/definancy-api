# did_parse — DefinancyDid (`did:definancy:{network}:{id}`)

> **Vectors:** `vectors/did_parse/` *(planned)*
> **Spec:** Definancy DID spec

## Algorithm

Trivial string composition.

## Java impl

- `languages/java/sdk/src/main/java/com/definancy/sdk/DID.java:22-24`
  `toString()` returns `"did:definancy:" + network + ":" + id.toString()`.

## TS impl

- `ts-definancy-sdk/src/identity/did.ts:22-24` `toString()` returns
  `` `did:definancy:${network}:${await id.toString()}` `` (async because
  `id.toString()` is async due to `sha512_256`).

## Divergence notes

- **No parser exists in either SDK.** Both classes are construct-and-format
  only. Conformance for "DID parse" implies future code; today the spec is
  format-only.
- **No validation of `network`:** neither side restricts the network token
  (no `[a-z0-9-]+` check, no length cap, no rejection of `:`). A network
  string containing `:` produces an ambiguous DID in both SDKs. Test
  vectors should pin known good networks (e.g., `dev`, `mainnet`).
- **TS `toString` is async**; Java is sync. No byte-level effect — just
  affects test ergonomics.
