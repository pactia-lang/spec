# Pactia Platform — platform packages

Version: **1.2**  
Status: **Specification**

Part of: [packages.md](packages.md) | [language-spec.md](language-spec.md)

Platform crates (e.g. `@pactia/rust-anb`, `@pactia/html-css-js`) are **ordinary packages** — same `pactia.toml` shape, resolution, registry rules, and lowering as `@pactia/kernel`. They publish **`export def @`** tags and **`export def #`** macros in `index.pactia`.

`@stack` is a **kernel tag** like `@guide` or `@topology` — optional product-scope profile fields. Platform defaults arrive through **macros and tags** from imported packages, not through compiler stack binding.

REST **`method`** and **`path`** on `@api` are ordinary tag fields from `@pactia/kernel` — not a separate wire layer.

---

## Product wiring

```pactia
import @pactia/kernel;
import @pactia/rust-anb;

product Relay {
  > B2B order relay between suppliers and retailers

  #rust_anb

  @topology { mode: microservices, }
}
```

| Piece | Role |
| --- | --- |
| `import @pactia/rust-anb;` | Brings that package's `export def` symbols into effectiveRegistry |
| `#rust_anb` | Product-scope **macro** — splices `@stack { … }` and other defs from the package body |
| `pactia.toml` `[dependencies]` | Semver range — same as any dependency |
| `pactia.lock` | Pinned version + digest |
| `@stack { … }` | Optional **kernel tag** — extra profile fields; lowers like any other `@` tag |

Product `pactia.toml`:

```toml
[package]
name = "relay"
version = "0.1.0"

[dependencies]
"@pactia/kernel" = "^1.0"
"@pactia/rust-anb" = "^1.0"
```

`#rust_anb` must resolve to a `export def #rust_anb` from an imported package. Unknown macro → `UNKNOWN_SYMBOL`.

Canonical example: [relay.pactia](https://github.com/pactia-lang/pactiac/blob/main/test/fixtures/kernel/relay.pactia).

### Legacy 1.1 binding (deprecated)

```pactia
@stack platform {
  #[rust_anb]
}
```

Compilers may accept this during transition. New products use product-level `#rust_anb` only.

---

## Platform package authoring

Author with **`export def`** in `index.pactia` — same as any crate's `lib.rs`:

```pactia
pactia 1.0
// Package: @pactia/rust-anb

export def #rust_anb in product {
  >> Rust / Actix / Node backend — primary APIs on actix-web and tokio. >>

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

Products invoke `#rust_anb` in `product { }` after `import @pactia/rust-anb;`. Registry merge follows [registry.md](registry.md).

---

## API wire fields

`@api` is a kernel tag. Wire-shaped fields such as `method` and `path` are **optional fields on the tag body** — validated from the `export def @api` spec in `@pactia/kernel`:

```pactia
import @pactia/kernel;

service OrderService {
  #list
  @@output OrderListResponse
  @api list_orders {
    method: GET,
    path: "/api/v1/orders",
  }
}
```

---

## Versions

| File | Role |
| --- | --- |
| `pactia.toml` | Semver **ranges** for dependencies |
| `pactia.lock` | **Pinned** version + digest (TOML) |

Bump the range, refresh `pactia.lock`, recompile. Review package release notes for macro or platform-law changes.

---

## See also

- [packages.md](packages.md)
- [compilation.md](compilation.md)
- [registry.md](registry.md)
