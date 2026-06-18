# Changelog

All notable specification changes are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Added

- **IR JSON Schema:** normative machine-readable schemas for module-scoped compiler output under `schemas/ir/` (`manifest`, `product`, module/model/service slices, full workspace bundle).
- **Kernel tag catalog:** normative `registry/kernel-tags.yaml` with keyword-aligned categories (`product`, `module`, `model`, `service`, `field`).
- **Tag body schemas:** stub JSON Schema files under `schemas/tags/*-v1.json` referenced by the kernel catalog.

### Changed

- **Normative fleet tag schemas:** `api-v1`, `auth-v1`, `entity-v1`, `stack-v1`, `public-v1`, `input-v1`, `output-v1`, `emit-v1`, `throws-v1`, `actor-v1`, and `deploy-v1` constrain bodies used in `fleet-management-v2.pactia`; other kernel tag schemas remain stubs.

- **Kernel registry model:** single `@surface` + `@bind` for multi-platform UI; no kernel `@web`, `@ios`, `@grpc`, or protocol/platform tags.
- **Service DTO tags:** kernel `@input` / `@output` replace `@body` / `@returns` (REST package may register aliases).
- **Service flags:** `#[database]`, `#[cache]`, `#[events]` macros only — no `@database` / `@cache` / `@events` kernel tags.
- **Package kind:** `domain` renamed to `vertical` for reusable model/service packages.
- **Registry categories:** aligned with keyword scope (`product` \| `module` \| `model` \| `service` \| `field`); `protocol` / `surface` / `compliance` are optional package `kind` metadata for IDE grouping only.
- **Docs:** remove legacy flat IR references (`project.yaml`, `domain.yaml`, `project-definition.yaml`, `domain.entities`); align lowering examples with module-scoped `*.model.yaml` paths.
- **Error taxonomy:** collapse stack-specific compiler codes into generic `PACKAGE_*` resolution errors; keep `STACK_BINDING_MISMATCH` for `@stack` tag + `pactia.toml [stack].package` agreement; rename `VERSION_IN_STACK` → `VERSION_IN_TAG_BODY`.
- **`@stack` in compiler:** treated as a kernel clause tag — same package resolver as `import`, no dedicated stack compile phase.
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
- Workspace registry: categories, selective `import { … } from @pkg`; `@stack` tag on `product` binds stack-kind package via generic package resolver.
- Tag system: clause / modifier / macro roles, scope matrix, prefix decorations, comma-separated fields, array literals.
- Cross-cutting blocks: `@policy`, `@security`, `@observe`, `@deploy`, `@guide`, `@must`, `@test`.
- Protocol packages: `@api` wire validation via `@pactia/protocol-rest`; `@grpc`, `@graphql` via protocol packages.
- Canonical reference: [fleet-management-v2.pactia](fixtures/kernel/fleet-management-v2.pactia).

[Unreleased]: https://github.com/pactia-lang/spec/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/pactia-lang/spec/releases/tag/v1.0.0
