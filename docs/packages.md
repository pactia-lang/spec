# Pactia Packages

Version: **1.2**

Pactia programs compose from **packages** — like Rust crates. Each package is `**pactia.toml` + `index.pactia`** only. There is **no** generated `pactia.package.json`.

Part of: [language-spec.md](language-spec.md) | [registry.md](registry.md) | [platform.md](platform.md)

Tag and macro catalogs live in **package source** (`index.pactia`). Baseline intent: `@pactia/kernel` from [pactia-lang/kernel](https://github.com/pactia-lang/kernel). Stack, protocol, surface, and vertical packages: [pactia-lang/pactia-io](https://github.com/pactia-lang/pactia-io).

---

## Crate mental model


| Rust                 | Pactia                                                                 |
| -------------------- | ---------------------------------------------------------------------- |
| `Cargo.toml`         | `pactia.toml` — name, version, kind, dependencies                      |
| `lib.rs`             | `index.pactia` — `export def @…` / `#…`                                |
| `cargo build`        | `pactia build` → invokes `pactiac compile`                             |
| crates.io / git deps | git remote + semver tag (Go-style); vendored under `.pactia/packages/` |


Authors edit `**index.pactia`**. The compiler parses it at product compile time and **derives IR slot metadata** from export def name + `in` placement (see [compilation.md](compilation.md#tag-lowering)).

---

## Package kinds


| `kind`             | Example                  | Role                                                                    |                                                |
| ------------------ | ------------------------ | ----------------------------------------------------------------------- | ---------------------------------------------- |
| `library`          | `@pactia/kernel`         | Domain-neutral tags and macros                                          |                                                |
| `@pactia/rust-anb` | `stack`                  |                                                                         | Platform law — `#stack_macro` at product scope |
| `protocol`         | `@pactia/protocol-rest`  | Wire validation on `@api` (see [Protocol packages](#protocol-packages)) |                                                |
| `vertical`         | `@pactia/kyc-compliance` | Domain patterns                                                         |                                                |
| `surface`          | `@pactia/html-css-js`    | UI / static-site stack registration                                     |                                                |


---

## Package manifest (`pactia.toml`)

### Library / stack / surface / vertical

```toml
[package]
name = "@pactia/rust-anb"
version = "1.0.0"
kind = "stack"

[dependencies]
"@pactia/kernel" = "^1.0"
```

### Protocol

Protocol packages may ship a **wire JSON Schema** referenced from TOML:

```toml
[package]
name = "@pactia/protocol-rest"
version = "1.0.0"
kind = "protocol"

[protocol]
wire-schema = "schemas/api-wire-v1.json"
```

When a product **imports** `@pactia/protocol-rest`, the compiler loads that schema from the vendored package directory and validates REST `**method`** and `**path**` fields on kernel `@api` blocks. Invalid wire → `WIRE_INVALID`.

Example schema (package-relative path):

```json
{
  "type": "object",
  "required": ["method", "path"],
  "properties": {
    "method": { "type": "string", "enum": ["GET", "POST", "PUT", "PATCH", "DELETE", "HEAD", "OPTIONS"] },
    "path": { "type": "string", "minLength": 1, "pattern": "^/" }
  }
}
```

Protocol packages do **not** require `index.pactia` when wire validation is schema-only. Stack and library packages require `index.pactia` with `export def`.

gRPC and GraphQL use sibling protocol packages with their own wire artifacts (proto, GraphQL schema, or JSON Schema).

---

## Product manifest (`pactia.toml`)

App / product workspaces use the same shape — like a binary crate:

```toml
[package]
name = "marketplace"
version = "0.1.0"

[dependencies]
"@pactia/kernel" = "^1.0"
"@pactia/protocol-rest" = "^1.0"
"@pactia/rust-anb" = "^1.0"
```


| Rule                 | Detail                                                                                        |
| -------------------- | --------------------------------------------------------------------------------------------- |
| No module list       | Multi-file products compose via **partial imports + attach** in `product.pactia`              |
| No `[stack]` section | Stack binding is `**#stack_macro` in source** after `import` — see [platform.md](platform.md) |
| Versions             | Only in `pactia.toml` / `pactia.lock` — never in `import` lines                               |


### `pactia.lock` (TOML)

```toml
lockVersion = 1

[[package]]
name = "@pactia/protocol-rest"
version = "1.0.0"
digest = "sha256:…"
```

Committed to git. Pins exact versions and content digests for vendored packages.

---

## Package authoring

`index.pactia` at package root — **no** `product` block:

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

Prose-first tags use `>` or `>> … >>` in def bodies; structured fields are optional.

---

## Import syntax

```pactia
import @pactia/kernel;
import { @api, #list } from @pactia/kernel;
import @pactia/protocol-rest;
import { orders, OrderService } from ./fragments/orders.pactia;
```


| Rule             | Detail                                        |
| ---------------- | --------------------------------------------- |
| Versions         | Only in `pactia.toml` / `pactia.lock`         |
| Registry symbols | Only from **imported** packages' `export def` |
| Local files      | Relative paths — fragment attach              |


---

## Package resolution

1. Read semver ranges from product `pactia.toml` `[dependencies]`
2. Pin exact versions from `pactia.lock`
3. Load vendored package from `.pactia/packages/@scope--name@version/` (or `PACTIA_VENDOR_ROOT`)
4. Parse `**index.pactia`** export defs into effectiveRegistry
5. For `kind: protocol`, load `**[protocol].wire-schema**` when validating `@api` wire fields

Stack tier: the package whose `**#stack_macro**` is invoked at product scope (e.g. `#rust_anb` → `@pactia/rust-anb`). That macro must be imported from the same coordinate.


| Code                      | Condition                                       |
| ------------------------- | ----------------------------------------------- |
| `PACKAGE_NOT_FOUND`       | Unknown coordinate                              |
| `PACKAGE_LOCK_MISMATCH`   | Digest mismatch                                 |
| `DEPENDENCY_NOT_DECLARED` | Import without `pactia.toml` entry              |
| `VERSION_IN_IMPORT`       | Semver in import line                           |
| `WIRE_INVALID`            | `@api` wire fields fail protocol package schema |
| `UNKNOWN_SYMBOL`          | `@` / `#` not in effectiveRegistry              |


---

## Publish and fetch

Packages publish as **git repositories** with semver tags — not a separate manifest build step.

```
my-stack/
  pactia.toml
  index.pactia
  schemas/          # protocol packages only
```

```bash
git tag v1.0.0 && git push origin v1.0.0
```

Consumers declare the dependency in `pactia.toml` and pin in `pactia.lock`. `pactia fetch` (planned) resolves git URL + tag/rev into `.pactia/packages/`.

Official repos:


| Repo                                                              | Packages                             |
| ----------------------------------------------------------------- | ------------------------------------ |
| [pactia-lang/kernel](https://github.com/pactia-lang/kernel)       | `@pactia/kernel`, `@pactia/kernel-*` |
| [pactia-lang/pactia-io](https://github.com/pactia-lang/pactia-io) | stack, protocol, surface, vertical   |


---

## Consumer compile

```bash
pactia build          # vendor lock → pactiac compile → out/
pactiac compile -w . -o out/
```

Imported defs merge into [effectiveRegistry](registry.md). Product files use `@`, `@@`, and `#` — never `export def`.

---

## CLI


| Command          | Role                                                |
| ---------------- | --------------------------------------------------- |
| `pactia build`   | Vendor deps, compile workspace                      |
| `pactia test`    | Compile workspace (acceptance harness TBD)          |
| `pactia init`    | (planned) `pactia.toml` + stub `product.pactia`     |
| `pactia add`     | (planned) Update toml + lock                        |
| `pactia fetch`   | (planned) Resolve git deps into `.pactia/packages/` |
| `pactia publish` | (planned) Tag + push package repo                   |


---

## See also

- [platform.md](platform.md) — stack macros and protocol imports
- [registry.md](registry.md) — effectiveRegistry and precedence
- [compilation.md](compilation.md) — IR lowering
- [language-spec.md](language-spec.md)

