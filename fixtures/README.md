# Fixtures

`.pactia` programs referenced by the specification.

Each fixture should compile with the pactiac version listed in `spec/README.md`.

## Current fixtures

| Path | Spec section |
| --- | --- |
| `kernel/fleet-management-v2.pactia` | [language-spec.md](../docs/language-spec.md) — full consumer product |
| `packages/fintech-rules-index.pactia` | [packages.md#package-authoring](packages.md#package-authoring) — `define macro` / `define tag` package source |

More fixtures will be added as normative sections cite minimal snippets.

## Layout

```
fixtures/
  kernel/
  packages/
  workspace/
```

Full runnable workspaces live in the [examples](https://github.com/pactia-lang/examples) repository.
