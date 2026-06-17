# Changelog

All notable specification changes are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

## [1.0.0] - TBD

First public language version.

### Added

- Human kernel: 11 keywords, `@tag { }`, `#[macro]`, `define template`, package `define tag` / `define macro`, `> prose`.
- Workspace layout for multi-module products.
- Package composition via Rust-style `use @scope/name;` — versions in `kabol.toml` / `kabol.lock` only.
- Workspace registry: categories, selective `use @pkg::{symbol};`, `@stack` + `use` for stack packages.
- Tag system: clause / modifier / macro roles, scope matrix, prefix decorations, comma-separated fields, array literals.
- Cross-cutting blocks: `@policy`, `@security`, `@observe`, `@deploy`, `@guide`, `@must`, `@test`.
- Protocol packages: `@rest`, `@grpc`, `@graphql` via `use @pactia/protocol-*`.
- Canonical reference: [fleet-management-v2.pactia](../examples/single-file/fleet-management-v2.pactia).

[Unreleased]: https://github.com/pactia-lang/spec/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/pactia-lang/spec/releases/tag/v1.0.0
