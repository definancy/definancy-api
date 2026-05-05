# api/

The Definancy OpenAPI wire contract.

## File

- `openapi.yaml` — OpenAPI 3.1 spec, hand-authored. The single source of
  truth for the wire shape (paths, components, request/response schemas,
  security schemes).

## Validation

Linted with [Spectral](https://stoplight.io/open-source/spectral) against
Definancy house rules. The ruleset lives in the consumer factory at
`tooling/lint/spectral.yaml`; the spec must pass `spectral lint` clean
on every PR (CI gate in `definancy-sdks`).

Breaking-change detection uses [oasdiff](https://www.oasdiff.com/)
against the last released `spec-v*` tag. Breaking diffs (in oasdiff's
ERR severity) block the PR.

## Rendering

The spec is auto-rendered as an interactive API reference on the hosted
docs site by [Scalar](https://scalar.com/) at release time. No manual
render step is required here — Scalar reads `openapi.yaml` directly.

For local inspection:

```bash
npx --yes @stoplight/spectral-cli lint openapi.yaml
```

…or open in any OpenAPI 3.1-aware tool (Stoplight Studio, Redocly,
Swagger UI ≥ 4.0, Insomnia).

## Scope boundary

This file describes the **wire contract only** — what bytes are sent and
received over HTTP. Algorithm-level behavior (DPoP proof construction,
canonical JSON, identity encoding, etc.) lives in
[`../conformance/`](../conformance/), not here. A new behavioral rule
goes in `conformance/algorithms/<name>.md` with a vector under
`conformance/vectors/<name>/`, not as descriptive prose in this file.
