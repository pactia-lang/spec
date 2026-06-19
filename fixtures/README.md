# Fixtures

`.pactia` programs referenced by the specification.

Each fixture should compile with the pactiac version listed in `spec/README.md`.

## Current fixtures

| Path | Spec section |
| --- | --- |
| `kernel/fleet-management-v2.pactia` | [language-spec.md](../docs/language-spec.md) — full consumer product |
| `kernel/fleet-management-mini.pactia` | [language-spec.md](../docs/language-spec.md) — compact tagged example |
| `kernel/fleet-management-prose.pactia` | [language-spec.md](../docs/language-spec.md) — prose-first example |
| `kernel/pactia-lang-website.pactia` | [language-spec.md](../docs/language-spec.md#multi-surface) — single-page marketing site |
| `packages/fintech-rules-index.pactia` | [packages.md](../docs/packages.md) — `export def @` / `export def #` package source |

More fixtures will be added as normative sections cite minimal snippets.

## Layout

```
fixtures/
  kernel/
  packages/
  workspace/
```

Full runnable workspaces live in the [examples](https://github.com/pactia-lang/examples) repository.
