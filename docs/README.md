# Specification documents

Normative and supporting documents for **Pactia 1.0** — an AI-native, model-agnostic intent language. Start with [overview.md](overview.md#three-altitudes).

Nine documents. Read top to bottom for a full picture.

## Documents

| Document | Description |
| --- | --- |
| [overview.md](overview.md) | Three altitudes, philosophy, intent line, architecture coverage |
| [language-spec.md](language-spec.md) | Kernel grammar, `define`, workspace layout, authorization |
| [grammar-reference.md](grammar-reference.md) | BNF and implementer error codes (not required for authors) |
| [registry.md](registry.md) | All `@tag { }`, `#[macro]`, cross-cutting blocks — skip if not using `import` |
| [packages.md](packages.md) | pactia.io packages, registry authoring, extensibility |
| [platform.md](platform.md) | Stack packages, semver / `kabol.lock`, protocol packages |
| [compilation.md](compilation.md) | Compiler pipeline, provenance, package build |
| [editor-support.md](editor-support.md) | VS Code / Cursor syntax highlighting |

## Examples

Runnable samples: [examples](https://github.com/pactia-lang/examples) repository.

Normative snippets: [../fixtures/](../fixtures/)
