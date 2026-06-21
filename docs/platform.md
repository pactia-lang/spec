# Pactia Platform — Stacks and Protocols

Version: **1.2**  
Status: **Specification**

Part of: [packages.md](packages.md) | [language-spec.md](language-spec.md)

Stack law and wire validation live in **packages** — `@pactia/kernel` (intent), stack packages (platform), protocol packages (wire). Published from [pactia-lang/kernel](https://github.com/pactia-lang/kernel) and [pactia-lang/pactia-io](https://github.com/pactia-lang/pactia-io).

---

## Selecting a stack

Stack packages are **imported** like any dependency. The product binds the stack with a **product-level stack macro** (e.g. `#rust_anb`) in source — **not** via a `[stack]` section in `pactia.toml`.

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
| `#rust_anb` | Stack macro at **product** scope — merges stack `@stack` profile and service macros |
| `pactia.toml` `[dependencies]` | Declares semver range for the stack package |
| `pactia.lock` | Pins exact version + digest |
| `@stack { … }` | **Optional** kernel tag — extra profile fields after the stack macro; not the binding mechanism |

Version ranges live in `pactia.toml` / `pactia.lock` — never in tag bodies.

Product `pactia.toml`:

```toml
[package]
name = "relay"
version = "0.1.0"

[dependencies]
"@pactia/kernel" = "^1.0"
"@pactia/protocol-rest" = "^1.0"
"@pactia/rust-anb" = "^1.0"
```

The stack macro invoked at product scope must come from an **imported** stack package. Unknown macro → `UNKNOWN_SYMBOL`.

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

Stack packages (`kind: stack`) publish platform law: language, framework, CI defaults, deployment baselines, and service macros.

Authoring: **`export def`** at file root in `index.pactia`:

```pactia
pactia 1.0
// Package: @pactia/rust-anb

export def #rust_anb in product {
  >> Rust / Actix / Node backend stack — primary APIs on actix-web and tokio. >>

  @stack {
    language: rust,
    framework: actix-web,
    …
  }
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

At product compile, the stack package merges platform law and registers macros per [registry.md](registry.md) precedence.

---

## Stack versions

| File | Role |
| --- | --- |
| `pactia.toml` | Semver **ranges** for stack + dependencies |
| `pactia.lock` | **Pinned** version + digest (TOML) |

```toml
lockVersion = 1

[[package]]
name = "@pactia/rust-anb"
version = "1.0.0"
digest = "sha256:…"
```

---

## Protocol packages

Protocol packages (`kind: protocol`) enable **wire validation** on kernel `@api` blocks.

### Import in product source

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

Importing `@pactia/protocol-rest` does not add new tags — it activates validation of **`method`** and **`path`** on `@api` using the package wire schema.

### Wire schema in `pactia.toml`

```toml
[package]
name = "@pactia/protocol-rest"
version = "1.0.0"
kind = "protocol"

[protocol]
wire-schema = "schemas/api-wire-v1.json"
```

The schema file ships beside `pactia.toml` in the published package (see [packages.md — Protocol packages](packages.md#protocol-packages)). The compiler resolves it from the vendored package directory at compile time.

| Field | Validated on | When |
| --- | --- | --- |
| `method` | `@api { method: … }` | Protocol package imported |
| `path` | `@api { path: … }` | Protocol package imported |

Failure → **`WIRE_INVALID`**.

gRPC and GraphQL use sibling packages (`@pactia/protocol-grpc`, …) with proto or schema artifacts under the same `[protocol]` convention.

The compiler lowers endpoints to JSON IR in the service slice — see [compilation.md](compilation.md).

---

## Migrating stack versions

Bump the range in `pactia.toml`, refresh `pactia.lock`, recompile. Review stack package release notes for macro or platform-law changes.

---

## See also

- [packages.md](packages.md)
- [compilation.md](compilation.md)
- [registry.md](registry.md)
