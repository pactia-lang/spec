# Changelog

All notable specification changes are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Added

- **`context` keyword** — package-relative file attachments; `export context`, attach `context(symbol)`, `def alias = context name { }`; pactiac lowers to `context[]`, pactia build indexes and bundles assets
- **Package manager docs** — `pactia install`, dual coordinates, `~/.pactia/config.toml`, lock-is-truth, `pactia update`, `pactia why`, `publish --dry-run` in [packages.md](docs/packages.md), [overview.md](docs/overview.md), [platform.md](docs/platform.md)
- **Crate model:** packages are `pactia.toml` + `index.pactia` only — no `pactia.package.json`; IR slots derived at product compile.
- **Official package repos:** [pactia-lang/kernel](https://github.com/pactia-lang/kernel), [pactia-lang/pactia-io](https://github.com/pactia-lang/pactia-io).
- **Go-style distribution:** packages resolve from **git remote + semver tag**; lockfile digest; vendored under `.pactia/packages/` — see [packages.md](docs/packages.md#go-modules-distribution).
- **`workspace.json`:** documented as single-file IR bundle for agents; slice files remain for per-scope use.

### Changed

- **Editor support:** document `context` keyword, blocks, attach syntax, and folding rules in [editor-support.md](docs/editor-support.md)
- **Editorial coherence pass:** README tooling table (spec 1.2 vs `pactia 1.0` version line), broken links, error-code alignment (`REGISTRY_COLLISION`), provenance table (`DEFINE`), lockfile schema note, cross-repo links, **removed tag-name-specific validation (state-graph codes and pass)**.
- **Spec coherence pass:** `README.md`, `docs/overview.md`, `docs/packages.md`, `docs/platform.md`, `docs/registry.md`, `docs/compilation.md`, `docs/language-spec.md`, `schemas/manifests/pactia-toml.schema.json` aligned to 1.2 crate model.
- **Removed stale references:** `pactia.package.json`, `[stack]` in product TOML, tarball fetch, pactia.io as sole registry.
- **Stack binding:** product-level `#macro` (e.g. `#rust-stack`) + stack `import` in source only — no `[stack].package` in `pactia.toml`.
- **Removed:** `schemas/manifests/pactia-package.schema.json` — obsolete `pactia.package.json` format dropped in 1.2 crate model.
- **Removed IR JSON Schema:** no `schemas/ir/` in this repo; compiler IR shape is prose in [compilation.md](docs/compilation.md) only.
- **Removed:** per-tag JSON Schema sidecars and tag-name routing tables — all tag bodies validate from package `export def` field specs only; lowering uses generic placement rules (`extensions[]`, `modifiers`, `fields[]`).
- **Removed:** tag-name-specific validation (e.g. state-graph cross-checks) from normative spec — one uniform field-spec path for every tag.
- **`pactia.toml`:** dropped `kind` field (Cargo-style — name, version, optional description, `[dependencies]` only).

### Changed (1.2 baseline)

- **Workspace assembly:** import + attach replaces automatic `modules/*` folder scan as the normative model; folder merge documented as legacy/deprecated.
- **Macro invocation:** `#name` / `#name(args)` replaces `#[name]` (legacy may still parse during transition).
- **Modifier invocation:** `@@output`, `@@pk`, etc. replace `@output` / `@pk` prefix on the line above hosts/fields.
- **Docs updated for 1.2:** `language-spec.md`, `compilation.md`, `platform.md`, `macros.md`, `registry.md`, `packages.md`, `grammar-reference.md`, `editor-support.md`, `overview.md`.
- **Removed `fixtures/`:** `.pactia` examples live in [pactiac test/fixtures](https://github.com/pactia-lang/pactiac/tree/main/test/fixtures) only — spec is docs + manifest schemas.

### Deprecated

- `#[macro]` bracket syntax — use `#macro`.
- `@stack { #[rust-stack] }` stack binding — use `#rust-stack` at product level.
- `modules/*/module.pactia` directory scan — use export + attach.

### Added (1.1 baseline)

- **Pactia 1.1 language model:** `def @` / `def #` with `in` placement; multiline prose `>> … >>`; module `def` constants; reserved words `view`, `interface`, `class`, `function`.
- **Tag names are package-owned:** `@api`, `@entity`, etc. live in **`@pactia/kernel`** on [pactia-lang/kernel](https://github.com/pactia-lang/kernel) — not in this spec repo.

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
- **`registry/kernel-tags.yaml` and `schemas/tags/`** — tag defs live in package `index.pactia`; `@pactia/kernel` on [pactia-lang/kernel](https://github.com/pactia-lang/kernel).
- **`schemas/ir/`** — removed; IR shape is documented in `docs/compilation.md` only.
- **`docs/stdlib/`** — removed; spec repo documents language and compiler mechanics only.

## [1.0.0] - TBD

Pre-1.1 specification (superseded by unreleased 1.1 pass).

### Added

- Three altitudes, `@tag { }`, `#[macro]`, workspace layout, package manager (`pactia.toml` / lockfile).
- Module-scoped IR, fleet fixtures.
- Protocol and stack packages model.

[Unreleased]: https://github.com/pactia-lang/spec/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/pactia-lang/spec/releases/tag/v1.0.0
