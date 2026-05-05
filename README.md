# definancy-spec

The Definancy contract — OpenAPI wire spec plus the cross-language
conformance suite that pins observable behavior. Single source of truth
for everything that needs to talk to Definancy on the wire.

## Layout

```
api/
  openapi.yaml          ← OpenAPI 3.1 wire contract
  README.md             ← spec render/lint specifics
conformance/
  algorithms/           ← markdown specs per algorithm domain
  vectors/              ← golden inputs + expected outputs
  README.md             ← schema, encoding suffixes, runner contract
CHANGELOG.md            ← release log (this file's tag chain)
```

## How to consume

Pin a tagged commit as a git submodule:

```bash
git submodule add https://github.com/definancy/definancy-spec.git spec
git -C spec checkout spec-v0.3.1
```

The wire shape (`api/openapi.yaml`) feeds OpenAPI code generators. The
conformance suite (`conformance/vectors/*.yaml`) feeds runners that
assert hand-written domain code produces byte-identical outputs across
languages — see [`conformance/README.md`](conformance/README.md) for the
runner contract.

The official Definancy SDKs are the visible reference consumers:

- [`definancy-sdk-typescript`](https://github.com/definancy/definancy-sdk-typescript)
- [`definancy-sdk-java`](https://github.com/definancy/definancy-sdk-java)

Each ships a conformance runner that exercises the full vector set on
every release.

## Releases

Releases are tagged `spec-v<MAJOR>.<MINOR>.<PATCH>`. See [CHANGELOG.md](CHANGELOG.md)
for the version history. Pre-1.0 releases use MINOR for any
contract-visible change (added paths, vector-value updates, layout
changes); PATCH is reserved for documentation-only updates.

Tags are immutable once pushed.

## Security

To report a vulnerability — including missing or incorrect conformance
vectors that fail to catch a real divergence — email
**security@definancy.com** rather than opening a public GitHub issue. We
acknowledge within 2 business days.

## License

See [LICENSE](LICENSE).
