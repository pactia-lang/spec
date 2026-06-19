# Pactia Platform — Stacks, Versions, and Protocols

Version: **1.1**  
Status: **Specification**

Part of: [packages.md](packages.md) | [language-spec.md](language-spec.md)

Wire tags (`@grpc`, REST wire fields) and stack symbols are defined in **packages** — typically `@pactia/kernel`, `@pactia/protocol-rest`, and stack packages. This document covers stack selection and protocol imports only.

---

## Selecting a stack

```pactia
product FleetManagement {
  > Fleet tracking platform

  @stack rust-anb { }
  @topology { mode: microservices, }
}
```

`@stack` is a package tag on `product` (from `@pactia/kernel` or the stack package). Bare id resolves to `@pactia/<id>`. Version comes from `pactia.toml` / `pactia.lock` — never from the tag body.

```toml
[stack]
package = "@pactia/rust-anb"

[dependencies]
"@pactia/kernel" = "^1.0"
"@pactia/rust-anb" = "^1.0"
```

Mismatch between `@stack` target, optional `import`, and `[stack].package` → `STACK_BINDING_MISMATCH`.

---

## Stack packages

Stack packages (`kind: stack`) publish platform law: language, framework, CI defaults, deployment baselines, and macro overrides.

Manifest: **`pactia.package.json`** (built from `index.pactia`).

Authoring: **`export def`** at file root in `index.pactia` — not YAML fragments.

```pactia
pactia 1.0
// Package: @pactia/rust-anb

export def #paginated_defaults in service {
  @cursor { default: 20, max: 100, }
}
```

`pactia package build` emits the manifest; `pactia publish` uploads the tarball.

At product compile, the stack package merges platform law and registers macros per [registry.md](registry.md) precedence.

---

## Stack versions

| File | Role |
| --- | --- |
| `pactia.toml` | Semver **ranges** for stack + dependencies |
| `pactia.lock` | **Pinned** version + digest (TOML) |

```toml
lock-version = 1

[[package]]
name = "@pactia/rust-anb"
version = "1.0.0"
digest = "sha256:…"
```

Products pin stacks the same way as any dependency.

---

## Protocol packages

Protocol packages (`kind: protocol`) register wire blocks nested inside `@api { }`.

```pactia
import @pactia/kernel;
import @pactia/protocol-rest;

service OrderService {
  @api create_order {
    method: POST,
    path: "/api/v1/orders",
  }
}
```

Import `@pactia/protocol-rest` enables validation of `method` and `path`. gRPC and GraphQL use sibling packages (`@pactia/protocol-grpc`, etc.) with nested tags defined in those packages.

The compiler lowers endpoints to JSON IR in the service slice — see [compilation.md](compilation.md).

---

## allowedProtocolPackages

Stack packages may declare which protocol coordinates a product may import. The compiler checks imports against that list when the stack manifest includes an `allowedProtocolPackages` entry (field name in stack package metadata).

---

## Migrating stack versions

Bump the range in `pactia.toml`, run `pactia build` to refresh `pactia.lock`, recompile. Review stack package release notes for macro or platform-law changes.

---

## See also

- [packages.md](packages.md)
- [compilation.md](compilation.md)
- [registry.md](registry.md)
