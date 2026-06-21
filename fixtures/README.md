# Fixtures

`.pactia` programs referenced by the specification.

Each fixture should compile with the pactiac version listed in `spec/README.md`.

## Current fixtures

| Path | Spec section | Notes |
| --- | --- | --- |
| `kernel/relay.pactia` | [platform.md](../docs/platform.md), [language-spec.md](../docs/language-spec.md) | **1.2 canonical** — monolith, `#rust_anb`, `#` / `@@` sigils |
| `kernel/pactia-lang-website.pactia` | [language-spec.md](../docs/language-spec.md#multi-surface) — single-page marketing site | 1.2 surface syntax |
| `packages/fintech-rules-index.pactia` | [packages.md](../docs/packages.md) — `export def @` / `export def #` package source | Package index only |

Legacy 1.1 fleet fixtures were removed from this tree; refreshed copies may return when migrated to 1.2. Until then see pactiac `test/fixtures/kernel/relay.pactia` and workspace relay.

Workspace attach example (not in this tree): [pactiac/test/fixtures/workspace/relay/](https://github.com/pactia-lang/pactiac/tree/main/test/fixtures/workspace/relay) in the pactiac repo.

More fixtures will be added as normative sections cite minimal snippets.

## Layout

```
fixtures/
  kernel/
  packages/
```

Full runnable workspaces live in the [examples](https://github.com/pactia-lang/examples) repository or pactiac `test/fixtures/workspace/`.
