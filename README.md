# definancy-spec

The Definancy contract — OpenAPI wire spec plus the cross-language
conformance suite that pins observable behavior. Single source of truth
for every consumer (the SDK factory, the daemon, future tools).

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

## How consumers use this repo

Pin a tagged commit as a git submodule:

```bash
git submodule add https://github.com/definancy/definancy-spec.git spec
git -C spec checkout spec-v0.3.0
```

The wire shape feeds code generators (`api/openapi.yaml`); the conformance
suite feeds runners that assert hand-written domain code produces
byte-identical outputs across languages (`conformance/vectors/*.yaml`).

Known consumers:
- [`definancy-sdks`](https://github.com/definancy/definancy-sdks) — multi-
  language SDK factory. Generates per-language SDKs from `api/openapi.yaml`
  and exercises them against `conformance/vectors/`.
- The Definancy daemon (private) — generates server stubs from the same
  OpenAPI spec and validates its own outputs against the same conformance
  vectors the SDKs use.

## Releases

Releases are tagged `spec-v<MAJOR>.<MINOR>.<PATCH>`. See [CHANGELOG.md](CHANGELOG.md)
for the version history. Pre-1.0 releases use MINOR for any contract-
visible change (added paths, vector-value updates, layout changes); PATCH
is reserved for documentation-only updates.

Tags are immutable once pushed.

## Contributing

The spec evolves via PRs to `main`. CI in the consumer factory
(`definancy-sdks`) lints with Spectral and runs `oasdiff` against the
last released tag, so breaking changes are flagged at PR time. Vector
changes are exercised by every consumer's conformance runner — a vector
that disagrees with one or more reference implementations blocks the
release.

## Security

Vulnerabilities — including missing or incorrect conformance vectors
that fail to catch a real divergence — should be reported per the
factory's [SECURITY.md](https://github.com/definancy/definancy-sdks/blob/main/SECURITY.md).

## License

See [LICENSE](LICENSE).
