# Pactia Packages

Version: **1.2**

Pactia programs compose from **packages**. Each package is **`pactia.toml` + `index.pactia`** only — **tags and macros** via `export def`. Nothing else.

Part of: [language-spec.md](language-spec.md) | [registry.md](registry.md) | [platform.md](platform.md)

| Repo | Packages |
| --- | --- |
| [pactia-lang/kernel](https://github.com/pactia-lang/kernel) | `@pactia/kernel`, `@pactia/kernel-*` |
| [pactia-lang/pactia-io](https://github.com/pactia-lang/pactia-io) | `@pactia/rust-anb`, `@pactia/html-css-js`, … |

---

## Mental models

### Rust crate (authoring)

| Rust | Pactia |
| --- | --- |
| `Cargo.toml` | `pactia.toml` — name, version, optional description, dependencies |
| `lib.rs` | `index.pactia` — `export def @…` / `#…` |
| `cargo build` | `pactia build` → invokes `pactiac compile` |

Authors edit **`index.pactia`**. The compiler parses export defs at product compile time and **derives IR slot metadata** from `in` placement and modifier flag only — no tag-name routing table (see [compilation.md](compilation.md#tag-lowering)).

### Go modules (distribution)

| Go | Pactia |
| --- | --- |
| Module path | Package coordinate (`@pactia/kernel`) |
| Version = git tag | Same |
| `go.sum` | `digest` in `pactia.lock` |
| Vendor cache | `.pactia/packages/@scope--name@version/` |
| `go get` | `pactia fetch` / `pactia add` (planned) |

**Import in source** uses the coordinate only:

```pactia
import @pactia/kernel;
```

Versions live in `pactia.toml` / `pactia.lock`. Any git host works.

---

## Package manifest (`pactia.toml`)

Cargo-style — **no `kind` field**. What a package exports lives in `index.pactia` only.

```toml
[package]
name = "@pactia/rust-anb"
version = "1.0.0"
description = "Rust / Actix platform macros"

[dependencies]
"@pactia/kernel" = "^1.0"
```

Product workspace:

```toml
[package]
name = "marketplace"
version = "0.1.0"

[dependencies]
"@pactia/kernel" = "^1.0"
"@pactia/rust-anb" = "^1.0"
```

| Rule | Detail |
| --- | --- |
| No module list | Import + attach in `product.pactia` |
| No `[stack]` in TOML | Platform binding is `#macro` in source — [platform.md](platform.md) |
| No `kind` | Unlike old drafts — all packages are equal crates; `@stack` is a tag in source |
| No extra TOML sections | Only `[package]` and `[dependencies]` |

### `pactia.lock`

```toml
lockVersion = 1

[[package]]
name = "@pactia/kernel"
version = "1.0.0"
digest = "sha256:…"
```

---

## Package authoring

```pactia
pactia 1.0
// Package: @acme/fintech-rules

export def @sanctions_check in service {
  > Enhanced screening intent — provider and level in prose or structured fields.
}

export def #sanctions_screen in service {
  @sanctions_check { level: enhanced, }
}
```

Prose-first: `>` or `>> … >>`. Structured fields optional.

---

## Import syntax

```pactia
import @pactia/kernel;
import { @api, #list } from @pactia/kernel;
import { orders, OrderService } from ./fragments/orders.pactia;
```

---

## Package resolution

1. Read ranges from `pactia.toml` `[dependencies]`
2. Pin from `pactia.lock`
3. Load vendored package from `.pactia/packages/` (or `PACTIA_VENDOR_ROOT`)
4. Parse **`index.pactia`** export defs into effectiveRegistry

All imported packages merge with the same rules. `@stack` lowers as a normal product-scope tag; `#rust_anb` expands as a normal product-scope macro.

| Code | Condition |
| --- | --- |
| `PACKAGE_NOT_FOUND` | Unknown coordinate |
| `PACKAGE_LOCK_MISMATCH` | Digest mismatch |
| `DEPENDENCY_NOT_DECLARED` | Import without `pactia.toml` entry |
| `VERSION_IN_IMPORT` | Semver in import line |
| `UNKNOWN_SYMBOL` | `@` / `#` not in effectiveRegistry |

---

## Publish and fetch

Git repo + semver tag. Ship **`pactia.toml` + `index.pactia`** only.

```bash
git tag v1.0.0 && git push origin v1.0.0
```

---

## Consumer compile

```bash
pactia build
pactiac compile -w . -o out/
```

---

## See also

- [platform.md](platform.md) — platform packages
- [registry.md](registry.md) — effectiveRegistry
- [compilation.md](compilation.md) — IR output
