# Pactia Platform — Stacks, Versions, and Protocols

Version: **1.2**  
Status: **Specification**

Part of: [packages.md](packages.md) | [language-spec.md](language-spec.md)

Wire tags (`@grpc`, REST wire fields) and stack symbols are defined in **packages** — typically `@pactia/kernel`, `@pactia/protocol-rest`, and stack packages. This document covers stack selection and protocol imports only.

---

## Selecting a stack

Stack packages are **imported** like any dependency. The product binds the stack with a **product-level stack macro** (e.g. `#rust_anb`) — not by nesting a macro inside `@stack { }`.

```pactia
import @pactia/kernel;
import @pactia/protocol-rest;
import @pactia/rust-anb;

product Relay {
  > B2B order relay between suppliers and retailers

  #rust_anb

  @topology { mode: microservices, }
}
```

| Piece | Role |
| --- | --- |
| `import @pactia/rust-anb;` | Brings stack `export def` symbols into the registry |
| `#rust_anb` | Stack-package macro at **product** scope — activates `@pactia/rust-anb` law |
| `[stack].package` in `pactia.toml` | Declares which stack coordinate the product binds |
| `pactia.lock` | Pins semver + digest |
| `@stack platform { … }` | **Optional** kernel tag — profile label + extra fields; **not** the binding mechanism |

Version ranges live in `pactia.toml` / `pactia.lock` — never in the tag body. Do **not** put `package: "@pactia/…"` fields inside `@stack`.

```toml
[stack]
package = "@pactia/rust-anb"

[dependencies]
"@pactia/kernel" = "^1.0"
"@pactia/protocol-rest" = "^1.0"
"@pactia/rust-anb" = "^1.0"
```

Mismatch between `#rust_anb` (or equivalent stack macro), `import @pactia/rust-anb`, and `[stack].package` → `STACK_BINDING_MISMATCH`.

Canonical example: [relay.pactia](../fixtures/kernel/relay.pactia).

### Legacy 1.1 binding (deprecated)

```pactia
@stack platform {
  #[rust_anb]
}
```

Compilers may accept this during transition. New products use product-level `#rust_anb` only.

---

## Stack packages

Stack packages (`kind: stack`) publish platform law: language, framework, CI defaults, deployment baselines, and macro overrides.

Manifest: **`pactia.package.json`** (built from `index.pactia`).

Authoring: **`export def`** at file root in `index.pactia` — not YAML fragments.

```pactia
pactia 1.0
// Package: @pactia/rust-anb

export def #rust_anb in product {
  > Rust / Actix / Node stack profile
}

export def #paginated in service {
  modifiers.pageSize: 50,
  modifiers.paginated: true,
}

export def #list in service {
  #paginated
}
```

Products activate the stack with `#rust_anb` in `product { }` after `import @pactia/rust-anb;`.

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
  #list
  @@output OrderListResponse
  @api list_orders {
    method: GET,
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
