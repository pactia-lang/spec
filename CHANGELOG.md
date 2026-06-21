# Changelog

All notable specification changes are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Added

- **Crate model:** packages are `pactia.toml` + `index.pactia` only — no `pactia.package.json`; IR slots derived at product compile.
- **Protocol wire schema:** `[protocol] wire-schema` in package `pactia.toml` (e.g. `@pactia/protocol-rest` → `schemas/api-wire-v1.json`); `WIRE_INVALID` on `@api` wire fields.
- **Official package repos:** [pactia-lang/kernel](https://github.com/pactia-lang/kernel), [pactia-lang/pactia-io](https://github.com/pactia-lang/pactia-io).
- **Go-style distribution:** packages resolve from **git remote + semver tag**; lockfile digest; vendored under `.pactia/packages/` — see [packages.md](docs/packages.md#go-modules-distribution).
- **`workspace.json`:** documented as single-file IR bundle for agents; slice files remain for per-scope use.

### Changed

- **Spec coherence pass:** `README.md`, `docs/overview.md`, `docs/packages.md`, `docs/platform.md`, `docs/registry.md`, `docs/compilation.md`, `docs/language-spec.md`, `schemas/manifests/pactia-toml.schema.json` aligned to 1.2 crate model.
- **Removed stale references:** `pactia.package.json`, `[stack]` in product TOML, tarball fetch, pactia.io as sole registry.
- **Stack binding:** product-level `#stack_macro` + stack `import` in source only — no `[stack].package` in `pactia.toml`.
- **`pactia-package.schema.json`:** marked deprecated (historical).

### Changed (1.2 baseline)

- **Workspace assembly:** import + attach replaces automatic `modules/*` folder scan as the normative model; folder merge documented as legacy/deprecated.
- **Macro invocation:** `#name` / `#name(args)` replaces `#[name]` (legacy may still parse during transition).
- **Modifier invocation:** `@@output`, `@@pk`, etc. replace `@output` / `@pk` prefix on the line above hosts/fields.
- **Docs updated for 1.2:** `language-spec.md`, `compilation.md`, `platform.md`, `macros.md`, `registry.md`, `packages.md`, `grammar-reference.md`, `editor-support.md`, `overview.md`.
- **Canonical fixture:** [relay.pactia](fixtures/kernel/relay.pactia) uses 1.2 syntax.

### Deprecated

- `#[macro]` bracket syntax — use `#macro`.
- `@stack { #[rust_anb] }` stack binding — use `#rust_anb` at product level.
- `modules/*/module.pactia` directory scan — use export + attach.

### Added (1.1 baseline)

- **Pactia 1.1 language model:** `def @` / `def #` with `in` placement; multiline prose `>> … >>`; module `def` constants; reserved words `view`, `interface`, `class`, `function`.
- **Tag names are package-owned:** `@api`, `@entity`, etc. live in **`@pactia/kernel`** on pactia.io — not in this spec repo.
- **IR JSON Schema:** normative schemas under `schemas/ir/` for compiler output (`.json` files).

### Changed (1.1 baseline)

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
