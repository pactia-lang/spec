# Changelog

All notable specification changes are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Changed

- **Kernel keyword:** rename `data` to `model` for domain modeling blocks (`model { @entity … }`).
- **Import & export:** single `import` keyword for registry packages and local files; remove `use`. Add `export` symbol modifier (replaces manifest-only `exports:`). `from` and `as` are import syntax tokens, not keywords.
- Lead with **three altitudes** (prose-only → light tagging → fully specified) in overview and language-spec.
- Collapse endpoint tags: **`@api`** is canonical; `@rest` / `@endpoint` deprecated (still parse).
- Rename error tags: **`@errors`** defines catalog; **`@throws`** references on `@api`.
- One modifier form per arity — remove `MODIFIER_TYPE_WRAPPER` and `{ type: X }` alternates.
- One prose rule: `>` required on every prose line (including altitude 0 in `product { }`).
- Split author vs implementer error codes; new [grammar-reference.md](docs/grammar-reference.md).
- Gate [registry.md](docs/registry.md) as opt-in for package `use` authors.

## [1.0.0] - TBD

First public language version.

### Added

- Human kernel: nine keywords, `@tag { }`, `#[macro]`, `define template`, package `define tag` / `define macro`, `> prose`.
- Workspace layout for multi-module products.
- Package composition via Rust-style `use @scope/name;` — versions in `kabol.toml` / `kabol.lock` only.
- Workspace registry: categories, selective `use @pkg::{symbol};`, `@stack` + `use` for stack packages.
- Tag system: clause / modifier / macro roles, scope matrix, prefix decorations, comma-separated fields, array literals.
- Cross-cutting blocks: `@policy`, `@security`, `@observe`, `@deploy`, `@guide`, `@must`, `@test`.
- Protocol packages: `@api`, `@grpc`, `@graphql` via `use @pactia/protocol-*`.
- Canonical reference: [fleet-management-v2.pactia](../fixtures/kernel/fleet-management-v2.pactia).

[Unreleased]: https://github.com/pactia-lang/spec/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/pactia-lang/spec/releases/tag/v1.0.0
