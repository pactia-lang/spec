# Pactia Platform — platform packages

Version: **1.2**  
Status: **Specification**

Part of: [packages.md](packages.md) | [language-spec.md](language-spec.md)

Platform crates (e.g. `@pactia/rust-stack`, `@pactia/html-css-js`) are **ordinary packages** — same `pactia.toml` shape, resolution, registry rules, and lowering as `@pactia/kernel`. They publish **`export def @`** tags and **`export def #`** macros in `index.pactia`.

Platform defaults arrive through **macros and tags** from imported packages, not through compiler stack binding. `@stack`, `@guide`, `@topology`, and every other name are ordinary package symbols — same validation and lowering as all tags.

Wire-shaped fields such as `method` and `path` are **optional fields on a tag body** when a package's `export def` declares them — not a separate wire layer in the language.

---

## Product wiring

```pactia
import @pactia/kernel;
import @pactia/rust-stack;

product Relay {
  > B2B order relay between suppliers and retailers

  #rust-stack

  @topology { mode: microservices, }
}
```

| Piece                          | Role                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------- |
| `import @pactia/rust-stack;`   | Brings that package's `export def` symbols into effectiveRegistry                     |
| `#rust-stack`                  | Product-scope **macro** — splices `@stack { … }` and other defs from the package body |
| `pactia.toml` `[dependencies]` | Semver range — same as any dependency                                                 |
| `pactia.lock`                  | Pinned version + digest                                                               |
| `@stack { … }`                 | Optional product-scope tag body — lowers like any other `@` host tag                  |

Product `pactia.toml`:

```toml
[package]
name = "relay"
version = "0.1.0"

[dependencies]
"@pactia/kernel" = "^1.0"
"@pactia/rust-stack" = "^1.0"
```

`#rust-stack` must resolve to a `export def #rust-stack` from an imported package. Unknown macro → `UNKNOWN_SYMBOL`.

Canonical example: [relay.pactia](https://github.com/pactia-lang/pactiac/blob/main/test/fixtures/kernel/relay.pactia).

### Legacy 1.1 binding (deprecated)

```pactia
@stack platform {
  #[rust-stack]
}
```

Compilers may accept this during transition. New products use product-level `#rust-stack` only.

---

## Platform package authoring

Author with **`export def`** in `index.pactia` — same as any crate's `lib.rs`:

```pactia
pactia 1.0
// Package: @pactia/rust-stack

export def #rust-stack in product {
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

Products invoke `#rust-stack` in `product { }` after `import @pactia/rust-stack;`. Registry merge follows [registry.md](registry.md).

---

## Wire fields on endpoints

When a package `export def` includes `method` and `path` as fields, authors may set them on any tag that uses that def — validated from the field spec only:

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

| File          | Role                               |
| ------------- | ---------------------------------- |
| `pactia.toml` | Semver **ranges** for dependencies |
| `pactia.lock` | **Pinned** version + digest (TOML) |

Bump the range, refresh `pactia.lock`, recompile. Review package release notes for macro or platform-law changes.

---

## See also

- [packages.md](packages.md)
- [compilation.md](compilation.md)
- [registry.md](registry.md)
