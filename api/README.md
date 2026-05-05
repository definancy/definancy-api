# api/

The Definancy OpenAPI wire contract.

## File

- `openapi.yaml` — OpenAPI 3.1 spec, hand-authored. The single source of
  truth for the wire shape (paths, components, request/response schemas,
  security schemes).

## Validation

The spec is linted with [Spectral](https://stoplight.io/open-source/spectral)
against Definancy house rules and checked for breaking changes against
the last released `spec-v*` tag with [oasdiff](https://www.oasdiff.com/).
Both gates run in the consuming projects' CI on every PR — a breaking
diff (oasdiff ERR severity) blocks the change.

For local lint:

```bash
npx --yes @stoplight/spectral-cli lint openapi.yaml
```

## Rendering

The spec is auto-rendered as an interactive API reference at release
time by [Scalar](https://scalar.com/), which reads `openapi.yaml`
directly — no manual render step.

For local inspection, open in any OpenAPI 3.1-aware tool (Stoplight
Studio, Redocly, Swagger UI ≥ 4.0, Insomnia).

## Scope boundary

This file describes the **wire contract only** — what bytes are sent and
received over HTTP. Algorithm-level behavior (DPoP proof construction,
canonical JSON, identity encoding, etc.) lives in
[`../conformance/`](../conformance/), not here. A new behavioral rule
goes in `conformance/algorithms/<name>.md` with a vector under
`conformance/vectors/<name>/`, not as descriptive prose in this file.
