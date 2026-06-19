# Pactia Packages & pactia.io

Version: **1.1**

Pactia programs compose from **packages** published to **pactia.io** (or a private registry). Stacks, verticals, and protocol wire formats are all packages.

Part of: [language-spec.md](language-spec.md) | [registry.md](registry.md)

Tag and macro catalogs live in **package source** (`index.pactia`) — not in this document. Typical baseline: `@pactia/kernel` on pactia.io.

---

## Why packages

| Without packages | With packages |
| --- | --- |
| Every product redefines KYC entities | `pactia add @pactia/kyc-compliance@^1.0` |
| Copy-paste integration blocks | `import @vendor/webhooks;` |

---

## Package kinds

| `kind` | Example | Role |
| --- | --- | --- |
| `stack` | `@pactia/rust-anb` | Platform law — bound on `product` via `@stack` |
| `vertical` | `@pactia/kyc-compliance` | Domain patterns merged into product |
| `protocol` | `@pactia/protocol-rest` | Wire tags nested in `@api` |
| `surface` | `@pactia/surface-react` | UI registration tags |

---

## Project files

### `pactia.toml`

```toml
[workspace]
entry = "product.pactia"
members = ["modules/commerce", "modules/identity"]

[stack]
package = "@pactia/rust-anb"

[dependencies]
"@pactia/kernel" = "^1.0"
"@pactia/protocol-rest" = "^1.0"
"@pactia/rust-anb" = "^1.0"
```

`[workspace]` is optional — omit for single-file products (discovery via `product.pactia` + imports).

### `pactia.lock` (TOML)

```toml
lock-version = 1

[[package]]
name = "@pactia/protocol-rest"
version = "1.0.0"
digest = "sha256:…"
```

Committed to git. Pins exact versions and tarball digests.

---

## Manifest: `pactia.package.json`

Built by `pactia package build` from `index.pactia`:

```json
{
  "name": "@acme/fintech-rules",
  "version": "1.0.0",
  "kind": "vertical",
  "registry": {
    "tags": [
      {
        "name": "sanctions_check",
        "in": ["service"],
        "ir": { "file": "service", "path": "endpoints[].sanctions", "merge": "merge_fields" }
      }
    ],
    "macros": [
      { "name": "sanctions_screen", "in": ["service"] }
    ]
  }
}
```

Authors edit **Pactia source**, not the manifest. `export def` drives registry exports; `pactia package build` adds **`ir`** slot metadata per tag (see [compilation.md](compilation.md#tag-lowering)).

---

## Package authoring

`index.pactia` at package root — **no** `product` block:

```pactia
pactia 1.0
// Package: @acme/fintech-rules

export def @sanctions_check in service {
  level,
  provider,
}

export def #sanctions_screen in service {
  @sanctions_check { level: enhanced, },
}
```

---

## Import syntax

```pactia
import @pactia/kyc-compliance;
import @pactia/kernel;
import "./local/overrides.pactia";
```

| Rule | Detail |
| --- | --- |
| Versions | Only in `pactia.toml` / `pactia.lock` — never in `import` |
| Registry symbols | Only from **imported** packages' `export def` (declare in `pactia.toml`, `import` in source) |
| Local files | Relative paths merge AST fragments |

---

## Dependencies vs import

| Layer | File | Role |
| --- | --- | --- |
| Declare | `pactia.toml` `[dependencies]` | Semver ranges |
| Pin | `pactia.lock` | Exact version + digest |
| Import | `.pactia` | Path only |

```bash
pactia add @pactia/kyc-compliance@^1.0
```

---

## Package resolution

Same resolver for `import @scope/name` and `@stack` target on `product`:

1. Read range from `pactia.toml`
2. Pin from `pactia.lock`
3. Fetch tarball; verify digest
4. Load `pactia.package.json` + parse `index.pactia` defs into effectiveRegistry

| Code | Condition |
| --- | --- |
| `PACKAGE_NOT_FOUND` | Unknown coordinate |
| `PACKAGE_LOCK_MISMATCH` | Digest mismatch |
| `DEPENDENCY_NOT_DECLARED` | Import without `pactia.toml` entry |
| `VERSION_IN_IMPORT` | Semver in import line |
| `STACK_BINDING_MISMATCH` | `@stack`, import, and `[stack].package` disagree |

---

## Package build

```
pactia package build -C ./packages/my-package
pactia publish -C ./packages/my-package
```

Pipeline:

1. Parse `index.pactia`
2. Collect `export def @` / `export def #` and compute `ir` slots
3. Write `pactia.package.json`
4. Validate; produce tarball digest

No per-tag JSON Schema files in the published bundle — field specs come from parsed `def` bodies.

---

## Consumer compile

```
pactia build          # resolve lock, compile product
pactiac compile -w . -o input/
```

Imported defs merge into [effectiveRegistry](registry.md). Product files use `@tag` and `#[macro]` — never `export def`.

---

## CLI

| Command | Role |
| --- | --- |
| `pactia init` | Create `pactia.toml` + stub `product.pactia` |
| `pactia add @scope/pkg@range` | Update toml + lock |
| `pactia package init` | Stub `index.pactia` + manifest |
| `pactia package build` | Emit `pactia.package.json` |
| `pactia publish` | Upload to registry |

---

## See also

- [registry.md](registry.md)
- [platform.md](platform.md)
- [compilation.md](compilation.md)
- [language-spec.md](language-spec.md)
