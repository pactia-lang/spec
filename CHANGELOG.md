# Changelog

All notable specification changes are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Added

- **Topology packages (1.3)** — structural exports for multi-team product composition. `export "./file.pactia"` manifest, three profiles (Registry/Topology/Mixed), `mixed-exports = true` opt-in, topology body inlining. See [language-spec.md — Topology packages](docs/language-spec.md#topology-packages-13), [packages.md — Topology](docs/packages.md#topology-packages-13), [grammar-reference.md](docs/grammar-reference.md).
- **Topology diagnostics** — `TOPOLOGY_DEF_FORBIDDEN`, `TOPOLOGY_WILDCARD_FORBIDDEN`, `PACKAGE_EXPORT_MIXED`, `EXPORT_NOT_DECLARED` error codes.
- **Package constants** — `export def name = value` in package `index.pactia`; resolvable via `import { name } from @pkg` and `${name}` interpolation in consumer prose and macro bodies. Bare `export name = value` (missing `def`) emits `CONSTANT_DEF_REQUIRED`. See [language-spec.md — Package constants](docs/language-spec.md#package-constants-12).
- **`CONSTANT_DEF_REQUIRED`** and **`EXPORT_KIND_AMBIGUITY`** diagnostic codes added to [grammar-reference.md](docs/grammar-reference.md#implementer-error-codes).
- **`effectiveRegistry.constants`** — package-exported constants merged into the compile-time symbol table alongside tags, macros, and contexts.
- **`context` keyword** — lowers to structural `context[]` on each IR slice; pactia build indexes and bundles assets
- **Package manager docs** — `pactia install`, dual coordinates, `~/.pactia/config.toml`, lock-is-truth, `pactia update`, `pactia why`, `publish --dry-run` in [packages.md](docs/packages.md), [overview.md](docs/overview.md), [platform.md](docs/platform.md)
- **Crate model:** packages are `pactia.toml` + `index.pactia` only — no `pactia.package.json`; IR slots derived at product compile.
- **Official package repos:** [pactia-lang/kernel](https://github.com/pactia-lang/kernel), [pactia-lang/pactia-io](https://github.com/pactia-lang/pactia-io).
- **Go-style distribution:** packages resolve from **git remote + semver tag**; lockfile digest; vendored under `.pactia/packages/` — see [packages.md](docs/packages.md#go-modules-distribution).
- **`workspace.json`:** documented as single-file IR bundle for agents; slice files remain for per-scope use.

### Changed

- **Source-order IR:** each slice lowers to ordered **`body[]`** (tags, prose, macro expansions) plus optional **`context[]`** (context keyword); `tag` names object shape — no per-type aggregation arrays.
- **`context[]` entries** use `name` (context block identifier), not `id`; `context.index.json` mirrors the same field.
- **`FRAGMENT_PACKAGE_IMPORT`** — warning when a fragment file contains a package import (assembly ignores it; product-level imports apply after merge).
- **Editor support:** document `context` keyword, blocks, attach syntax, and folding rules in [editor-support.md](docs/editor-support.md)
- **Editorial coherence pass:** README tooling table (spec 1.2 vs `pactia 1.0` version line), broken links, error-code alignment (`REGISTRY_COLLISION`), provenance table (`DEFINE`), lockfile schema note, cross-repo links, **removed tag-name-specific validation (state-graph codes and pass)**.
- **Spec coherence pass:** `README.md`, `docs/overview.md`, `docs/packages.md`, `docs/platform.md`, `docs/registry.md`, `docs/compilation.md`, `docs/language-spec.md`, `schemas/manifests/pactia-toml.schema.json` aligned to 1.2 crate model.
- **Removed stale references:** `pactia.package.json`, `[stack]` in product TOML, tarball fetch, pactia.io as sole registry.
- **Stack binding:** product-level `#macro` (e.g. `#rust-stack`) + stack `import` in source only — no `[stack].package` in `pactia.toml`.
- **Removed:** `schemas/manifests/pactia-package.schema.json` — obsolete `pactia.package.json` format dropped in 1.2 crate model.
- **Removed IR JSON Schema:** no `schemas/ir/` in this repo; compiler IR shape is prose in [compilation.md](docs/compilation.md) only.
- **Removed:** per-tag JSON Schema sidecars — all tag bodies validate from package `export def` field specs only; lowering uses source-order `body[]` per slice (`modifiers` merge into next host; field modifiers on model lines).
- **Removed:** tag-name-specific validation (e.g. state-graph cross-checks) from normative spec — one uniform field-spec path for every tag.
- **`pactia.toml`:** dropped `kind` field (Cargo-style — name, version, optional description, `[dependencies]` only).

### Changed (1.2 baseline)

- **Workspace assembly:** import + attach in `product.pactia` is normative; folder layout is convention only.
- **Macro invocation:** `#name` / `#name(args)` replaces `#[name]` (bracket syntax removed in 1.2).
- **Modifier invocation:** `@@output`, `@@pk`, etc. replace `@output` / `@pk` prefix on the line above hosts/fields.
- **Docs updated for 1.2:** `language-spec.md`, `compilation.md`, `platform.md`, `macros.md`, `registry.md`, `packages.md`, `grammar-reference.md`, `editor-support.md`, `overview.md`.
- **Removed `fixtures/`:** `.pactia` examples live in [pactiac test/fixtures](https://github.com/pactia-lang/pactiac/tree/main/test/fixtures) only — spec is docs + manifest schemas.

### Deprecated

- `#[macro]` bracket syntax — use `#macro`.
- `@stack { #[rust-stack] }` stack binding — use `#rust-stack` at product level.
- `modules/*/module.pactia` fixed-filename scan — use `<name>.module.pactia` export layout under `modules/<name>/`.

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

Pre-1.1 specification (superseded by 1.2).

### Added

- Three altitudes, `@tag { }`, `#[macro]`, workspace layout, package manager (`pactia.toml` / lockfile).
- Module-scoped IR, fleet fixtures.
- Protocol and stack packages model.

[Unreleased]: https://github.com/pactia-lang/spec/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/pactia-lang/spec/releases/tag/v1.0.0
