# Pactia Packages

Version: **1.2** (updated for 1.4 — multi-file packages, file-local imports)

Pactia programs compose from **packages**. Each package is **`pactia.toml` + `index.pactia`** plus optional internal `.pactia` files referenced via `export "./file"` — **tags and macros** via `export def`. Package internal files declare their own imports (see [language-spec.md — File-local imports](language-spec.md#file-local-imports)).

Part of: [language-spec.md](language-spec.md) | [registry.md](registry.md) | [platform.md](platform.md)

| Repo                                                              | Packages                                       |
| ----------------------------------------------------------------- | ---------------------------------------------- |
| [pactia-io](https://github.com/pactia-io)                         | `@pactia/kernel`, `@pactia/rust-stack`, …      |
| Any git host (via coordinate)                                     | `@github.com/org/repo`, `@gitlab.com/org/repo` |

---

## Commands

```bash
pactia init <dir> [--name <ProductName>]
pactia add <coordinate> [range] [-C <workspace>]
pactia install [-C <workspace>]
pactia update [<coordinate>] [-C <workspace>]
pactia build [-C <workspace>] [-o <output-dir>]
```

| Command   | Role                                                                 |
| --------- | -------------------------------------------------------------------- |
| `init`    | Minimal `pactia.toml` + `product.pactia` (prose only; no lock)       |
| `add`     | Add a dependency range, resolve lock, download, vendor               |
| `install` | Read `pactia.lock`, download pinned versions, vendor (no re-resolve) |
| `update`  | Re-resolve ranges from `pactia.toml`, refresh lock, download, vendor |
| `build`   | `install` from lock, then `pactiac compile`                          |
| `why`     | Show why a locked package is in the dependency graph                   |
| `publish` | `publish --dry-run` validates `pactia.toml` + `index.pactia` before tagging |
| `outdated` | Show deps with newer versions available. Accepts `--json`. |
| `clean` | Remove vendor dir + build output |

Global flags: `--help` / `-h`, `--version` / `-v`, `--json` (machine-readable output on `outdated`, `why`, `build`).

Default dependency range on `add` is `^1.0`.

**Lock is truth** for `install` and `build`: versions come from `pactia.lock` only. Re-resolve on `add` or `update`. Digest mismatches fail with a clear error.

Typical flows:

```bash
# New workspace
pactia init my-product --name MyProduct
# edit product.pactia — imports, #rust-stack, modules
pactia add @pactia/kernel
pactia add @pactia/rust-stack
pactia build -C my-product

# Clone existing repo
pactia install -C my-product
pactia build -C my-product

# Refresh pins after bumping ranges in pactia.toml
pactia update -C my-product
```

---

## Coordinates

Two equivalent styles:

**Shorthand scope** (curated prefix — configured in `~/.pactia/config.toml`):

```pactia
import @pactia/kernel;
```

```toml
"@pactia/kernel" = "^1.0"
```

**Go-style host path** (self-describing — git host in the coordinate):

```pactia
import @github.com/acme/fleet-rules;
```

```toml
"@github.com/acme/fleet-rules" = "^1.0"
```

Short names on the CLI resolve to `@pactia/{name}` (e.g. `pactia add rust-stack` → `@pactia/rust-stack`).

Invalid: `@github/package` or `@github.com/foo` — use full `@host/org/repo`.

Vendor directory encodes `/` as `--`: `@github.com/acme/fleet-rules@1.0.0` → `.pactia/packages/@github.com--acme--fleet-rules@1.0.0/`.

---

## Config (`~/.pactia/config.toml`)

Machine-local. Not in the product `pactia.toml`. The pactia CLI reads **all** git bases, API URLs, and download preferences from this file.

First install copies `config/config.example.toml` from the [pactia](https://github.com/pactia-lang/pactia) repo when the file is missing.

```toml
[source."@pactia/"]
git = "https://github.com/pactia-io"

[hosts."github.com"]
git = "https://github.com"
api = "https://api.github.com"
token = "env:PACTIA_GITHUB_TOKEN"

[hosts."gitlab.com"]
git = "https://gitlab.com"
api = "https://gitlab.com/api/v4"
token = "env:PACTIA_GITLAB_TOKEN"

[defaults]
prefer = "http"    # HTTP archive first; git clone fallback
```

| Entry | Derivation |
| ----- | ---------- |
| `[source."@pactia/"]` | `@pactia/{name}` → `{git}/{name}` at tag `v{version}` |
| `[hosts."github.com"]` | `@github.com/{org}/{repo}` → `{git}/{org}/{repo}` |
| `[hosts."gitlab.com"]` | `@gitlab.com/{org}/{repo}` → `{git}/{org}/{repo}` |

Missing `[hosts."…"]` or `[source."…"]` for a coordinate → configure `config.toml` (clear error).

Set `PACTIA_VENDOR_ROOT` to a local package directory during development (monorepo fixtures).

---

## Mental models

### Rust crate (authoring)

| Rust          | Pactia                                                            |
| ------------- | ----------------------------------------------------------------- |
| `Cargo.toml`  | `pactia.toml` — name, version, optional description, dependencies |
| `lib.rs`      | `index.pactia` — `export def @…` / `#…`                           |
| `cargo build` | `pactia build` → invokes `pactiac compile`                        |

Authors edit **`index.pactia`**. The compiler validates from field specs and placement, then lowers each scope to an ordered **`body[]`** in source sequence (see [compilation.md](compilation.md#source-order-body)).

### Go modules (distribution)

| Go                | Pactia                                   |
| ----------------- | ---------------------------------------- |
| Module path       | Package coordinate (`@pactia/kernel`)    |
| Version = git tag | Same (`v1.0.0`, `v1.0.0-beta.1`, …)      |
| `go.sum`          | `digest` in `pactia.lock`                |
| Vendor cache      | `.pactia/packages/@scope--name@version/` |
| `go get`          | `pactia add` / `pactia update`           |

**Import in source** uses the coordinate only:

```pactia
import @pactia/kernel;
```

Versions live in `pactia.toml` / `pactia.lock`. Any configured git host works.

---

## Package manifest (`pactia.toml`)

Cargo-style — **no `kind` field**. What a package exports lives in `index.pactia` only.

```toml
[package]
name = "@pactia/rust-stack"
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
"@pactia/rust-stack" = "^1.0"
```

| Rule                   | Detail                                                                         |
| ---------------------- | ------------------------------------------------------------------------------ |
| No module list         | Import + attach in `product.pactia` (folder-agnostic fragment paths)           |
| No `[stack]` in TOML   | Platform binding is `#macro` in source — [platform.md](platform.md)            |
| No `kind`              | Unlike old drafts — all packages are equal crates; `@stack` is a tag in source |
| No extra TOML sections | Only `[package]` and `[dependencies]`                                          |

### Export profile fields

Topology packages may declare their export profile in `pactia.toml`:

```toml
[package]
name = "@acme/marketplace-backend"
version = "1.0.0"
exports = "topology"       # "registry" (default) | "topology"
```

| Field | Values | Meaning |
|-------|--------|---------|
| `exports` | `"registry"` (default), `"topology"` | Declared export profile; validated against `index.pactia` content (`PACKAGE_PROFILE_MISMATCH`) |
| `mixed-exports` | `true` / `false` (default) | Opt-in for packages exporting both registry defs and topology symbols; discouraged (`HYBRID_PACKAGE_DISCOURAGED` warning) |

When `mixed-exports = true`, the `exports` field is ignored. Preferred: two separate coordinates (one registry, one topology).

### `pactia.lock`

Required when the product **imports packages**. Optional for altitude-0 products with no `@scope/name` imports (no vendored deps to pin).

```toml
lockVersion = 1

[[package]]
name = "@pactia/kernel"
version = "1.0.0"
digest = "sha256:…"
```

Digest covers `pactia.toml` + `index.pactia` + manifest-referenced `export "./…"` files in the vendored tree (or `.digest` marker when present).

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

export def max_page = 100
export def hint = > Validate all inputs.
```

Prose-first: `>` or `>> … >>`. Structured fields optional.

**Package constants** (`export def name = value`): exported named values (literals or prose) resolved at compile time from vendored `index.pactia`. Consumers use `import { name } from @pkg` to import constants. `export name = value` without `def` emits `CONSTANT_DEF_REQUIRED`.

---

## Import syntax

```pactia
import @pactia/kernel;
import { @api, #list } from @pactia/kernel;
import { orders, OrderService } from ./fragments/orders.pactia;
```

---

## Package resolution

**At compile time (`pactiac`):**

1. Read pinned versions from `pactia.lock`
2. Load vendored package from `.pactia/packages/` (or `PACTIA_VENDOR_ROOT`)
3. Parse **`index.pactia`** export defs into effectiveRegistry

**At package-manager time (`pactia`):**

- `add` / `update` — resolve semver ranges, write `pactia.lock`, download to `~/.pactia/packages/`, copy to `.pactia/packages/`
- `install` / `build` — use lock pins only; verify digests

All imported packages merge with the same rules. `@stack` lowers as a normal product-scope tag; `#rust-stack` expands as a normal product-scope macro.

| Code                      | Condition                                      |
| ------------------------- | ---------------------------------------------- |
| `PACKAGE_NOT_FOUND`       | Unknown coordinate                             |
| `LOCK_DIGEST_MISMATCH`    | Vendored tree digest ≠ lock                    |
| `LOCK_STALE`              | `pactia.toml` and lock out of sync             |
| `LOCK_MISSING`              | Dependencies declared but no `pactia.lock`     |
| `DEPENDENCY_NOT_DECLARED` | Import without `pactia.toml` entry             |
| `VERSION_IN_IMPORT`       | Semver in import line                          |
| `UNKNOWN_SYMBOL`          | `@` / `#` not in effectiveRegistry             |

---

## Topology packages (1.3)

Packages may export structural topology (modules, services, models, contexts) for multi-team product composition. See [language-spec.md — Topology packages](language-spec.md#topology-packages-13).

**Manifest (`index.pactia`):**
```pactia
pactia 1.0
export "./commerce.module.pactia"
export "./orders.service.pactia"
export "./orders.model.pactia"
```

**Bare topology file:**
```pactia
export module commerce {
  service OrderApi { @api list { method: GET, path: "/orders" } }
}
```

**Consumer:**
```pactia
import { commerce, OrderService } from @acme/marketplace-backend;
product MyApp { module(commerce) { service(OrderService) { } } }
```

**Profiles:** `detectPackageProfile()` determines Registry, Topology, or Mixed from `index.pactia` content. Mixed packages require `mixed-exports = true` in `pactia.toml [package]`.

`export "./file"` works for both topology and registry exports — package authors can split `export def @/@@/#` and `export def name = value` across multiple files referenced from `index.pactia`. Internal files may import packages independently.

---

## Publish

Git repo + semver tag. Ship **`pactia.toml` + `index.pactia`** plus all files referenced via `export "./file"` in the repo tree. Digest covers the full file closure.

```bash
git tag v1.0.0 && git push origin v1.0.0
```

Validate before tagging:

```bash
pactia publish --dry-run -C .
```

Pre-release tags (`v1.0.0-beta.1`, …) are valid. Semver ranges decide what satisfies a dependency.

---

## Consumer compile

```bash
pactia install    # or rely on pactia build to install from lock first
pactia build
pactiac compile -w . -o input/
```

---

## See also

- [platform.md](platform.md) — platform packages
- [registry.md](registry.md) — effectiveRegistry
- [compilation.md](compilation.md) — IR output
