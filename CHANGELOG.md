# Changelog

All notable specification changes are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Changed

- **Compiler IR layout:** `manifest.yaml` (version, entry, lockfile digest, module file index, `references[]`); `product.yaml` (includes surfaces, security, deployment); module slices `<module>.module.yaml`, `<module>.model.yaml`, `<service>.service.yaml`.
- **Kernel keyword:** `model` for domain modeling blocks (`model { @entity … }`).
- **Import & export:** single `import` keyword for registry packages and local files; `export` symbol modifier in package source (manifest `exports:` derived at build).
- Lead with **three altitudes** (prose-only → light tagging → fully specified) in overview and language-spec.
- **`@api`** is the kernel endpoint clause; `@pactia/protocol-rest` adds REST wire validation (`method`, `path`).
- **`@errors`** defines catalog; **`@throws`** references on `@api`.
- One modifier form per arity — no `{ type: X }` wrapper alternates.
- One prose rule: `>` required on every prose line (including altitude 0 in `product { }`).
- Split author vs implementer error codes; [grammar-reference.md](docs/grammar-reference.md) for implementers.
- Gate [registry.md](docs/registry.md) as opt-in for package authors and protocol wire validation.
- **Package manager:** `pactia` CLI with `pactia.toml` / `pactia.lock` (Cargo model).

## [1.0.0] - TBD

First public language version.

### Added

- Human kernel: nine keywords, `@tag { }`, `#[macro]`, `define template`, package `define tag` / `define macro`, `> prose`.
- Workspace layout for multi-module products.
- Package composition via `import @scope/name;` — versions in `pactia.toml` / `pactia.lock` only.
- Workspace registry: categories, selective `import { … } from @pkg`, `@stack` + `import` for stack packages.
- Tag system: clause / modifier / macro roles, scope matrix, prefix decorations, comma-separated fields, array literals.
- Cross-cutting blocks: `@policy`, `@security`, `@observe`, `@deploy`, `@guide`, `@must`, `@test`.
- Protocol packages: `@api` wire validation via `@pactia/protocol-rest`; `@grpc`, `@graphql` via protocol packages.
- Canonical reference: [fleet-management-v2.pactia](fixtures/kernel/fleet-management-v2.pactia).

[Unreleased]: https://github.com/pactia-lang/spec/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/pactia-lang/spec/releases/tag/v1.0.0
