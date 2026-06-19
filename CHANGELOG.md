# Changelog

All notable specification changes are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Added

- **Pactia 1.1 language model:** `def @` / `def #` with `in` placement; multiline prose `>> … >>`; module `def` constants; reserved words `view`, `interface`, `class`, `function`.
- **`[workspace]` in `pactia.toml`:** optional entry + members — no separate workspace manifest file.
- **Tag names are package-owned:** `@api`, `@entity`, `#[list]`, etc. live in **`@pactia/kernel`** on pactia.io — not in this spec repo.
- **IR JSON Schema:** normative schemas under `schemas/ir/` for compiler output (`.json` files).

### Changed

- **Language spec is language-only:** no tag/macro catalog in `language-spec.md`; names come from imported packages (typically `@pactia/kernel`).
- **Registry doc is mechanics-only:** resolution, tiers, precedence — not a tag encyclopedia.
- **Compiler IR format:** JSON only (`*.module.json`, `*.model.json`, `*.service.json`, `manifest.json`, `product.json`).
- **Package manifest:** `pactia.package.json` (not YAML).
- **Lockfile:** TOML `pactia.lock` (not YAML).
- **Tag validation:** field specs from parsed `def` bodies — no `schemas/tags/*` sidecar files.
- **Macros:** C-style call-site substitution; `in` placement (replaces `category` / decorator model).
- **Docs rewritten:** `compilation.md`, `packages.md`, `platform.md`, `registry.md`, `grammar-reference.md`, `macros.md`, `overview.md`.

### Removed

- **`define tag` / `define macro` / `define template`** — replaced by `def`.
- **`yaml merge` / `yaml package/*` embeds.**
- **`pactia.workspace.yaml` / `pactia-workspace.schema.json`.**
- **`registry/kernel-tags.yaml` and `schemas/tags/`** — tag defs live in package `index.pactia`; `@pactia/kernel` on pactia.io.
- **`docs/stdlib/`** — removed; spec repo documents language and compiler mechanics only.

## [1.0.0] - TBD

Pre-1.1 specification (superseded by unreleased 1.1 pass).

### Added

- Three altitudes, `@tag { }`, `#[macro]`, workspace layout, package manager (`pactia.toml` / lockfile).
- Module-scoped IR, state graph validation, fleet fixtures.
- Protocol and stack packages model.

[Unreleased]: https://github.com/pactia-lang/spec/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/pactia-lang/spec/releases/tag/v1.0.0
