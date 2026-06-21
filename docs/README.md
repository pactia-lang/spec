# Specification documents

Normative and supporting documents for **Pactia 1.2** — an AI-native, model-agnostic intent language. Start with [overview.md](overview.md#three-altitudes).

## Documents

| Document | Description |
| --- | --- |
| [overview.md](overview.md) | Three altitudes, philosophy, intent line, architecture coverage |
| [language-spec.md](language-spec.md) | **Language only** — grammar, blocks, `def`, `@` / `@@` / `#` sigils, attach, migration from 1.1 |
| [macros.md](macros.md) | **Normative** — unified `def` for tags and macros |
| [grammar-reference.md](grammar-reference.md) | BNF and implementer error codes |
| [registry.md](registry.md) | Symbol resolution, tiers, precedence — **not** a tag catalog |
| [packages.md](packages.md) | `pactia.toml`, imports, publish, `export def` |
| [compilation.md](compilation.md) | Compiler pipeline, JSON IR layout, tag lowering, provenance |
| [platform.md](platform.md) | Stack and protocol packages |
| [editor-support.md](editor-support.md) | VS Code / Cursor syntax highlighting |

Tag and macro **names** (e.g. `@entity`, `#list`, `@@output`) live in packages such as **`@pactia/kernel`** on pactia.io — not in this spec tree.

## Examples

Runnable samples: [examples](https://github.com/pactia-lang/examples) repository.

Normative snippets: [../fixtures/](../fixtures/)
