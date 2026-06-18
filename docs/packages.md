# Pactia Packages & pactia.io

Pactia programs compose from **packages** — published to **pactia.io** and resolved the same way regardless of kind. Everything is a package: vertical patterns, protocol wire formats, and tech stacks.

Part of the intent-first ecosystem: BSC vision | [language-spec.md](language-spec.md)

---

## Why packages

| Without packages                          | With packages                            |
| ----------------------------------------- | ---------------------------------------- |
| Every P2P exchange redefines KYC entities | `pactia add @pactia/kyc-compliance@^1.0` then `import @pactia/kyc-compliance;` |
| Every marketplace redefines dispute flow  | `import { … } from @pactia/dispute-resolution;` selective import |
| Copy-paste integration blocks             | `import @vendor/sumsub-webhooks;` |

Architects write product-specific Pactia; community/vendor packages supply commodity patterns.

---

## Package kinds

| `kind` | Coordinate example | Selected via | Typical exports |
| --- | --- | --- | --- |
| **`stack`** | `@pactia/rust-anb` | `@stack { }` on `product` | Platform law — language, framework, CI, platformLaw |
| **`vertical`** | `@pactia/kyc-compliance` | `import @scope/name` | Entities, rules, integrations, optional services |
| **`protocol`** | `@pactia/protocol-rest` | `import @scope/name` | Expose block name + schema + YAML lowering |
| **`vertical`** (financial) | `@pactia/escrow-custody` | `import @scope/name` | EscrowHold entity, custody integration template |
| **`vertical`** (marketplace) | `@pactia/marketplace-core` | `import @scope/name` | Offer, Order, listing services |
| **`vertical`** (vendor) | `@sumsub/pactia-webhooks` | `import @scope/name` | Official Sumsub → Pactia integration |
| **`vertical`** (org-private) | `@acme/internal-auth` | `import @scope/name` | Company-specific roles (private registry) |

All kinds use the same registry, manifest format, semver, and `pactia.lock`. The `kind` field drives how the compiler processes the package:
- `stack` — merged as platform law; selected once per `product`
- `vertical` — declarations merged into compilation unit
- `protocol` — registers expose block names; see [platform.md](platform.md#protocol-packages)

Vertical packages **cannot** override stack baselines (`platformLaw`, `technologyPolicy`). They export **intent**, not platform law.

---

## Manifest: `pactia.package.yaml`

Every publishable package includes:

```yaml
package:
  name: "@pactia/kyc-compliance"
  version: 1.0.0
  description: Reusable KYC status model and verification webhook
  license: MIT
  kind: vertical
  pactiaVersion: "1.0"
  compatibleStacks:
    - name: "@pactia/rust-anb"
      version: "^1.0"
  repository: https://github.com/pactia-io/kyc-compliance
  # exports: derived at build from `export` modifiers in source — not hand-authored
  registry:
    tags:
      - name: sanctions_check
        category: service
        kind: compliance
        version: 1
      - name: kyc_status_field
        category: model
        version: 1
    macros:
      - name: sanctions_screen
        category: service
        kind: compliance
        version: 1
    reexports: []
  dependencies:
    - package: "@pactia/audit-trail"
      version: "^1.0"
```

### Export sections (derived)

At **`pactia package build`**, the compiler scans `export` modifiers on symbols in package source and writes manifest export flags. Authors do **not** hand-edit `exports:` in `pactia.package.yaml`.

| `export` on | Merged into consumer when imported |
| --- | --- |
| `@entity`, `@enum`, shapes in `model { }` | `modules/<module>/<module>.model.yaml` entities, enums, invariants |
| `@rule { }` blocks | `<module>.module.yaml` rules |
| `@integration { }` blocks | `<module>.module.yaml` integrations |
| `@actor { }` blocks | `<module>.module.yaml` actors |
| `service` blocks | `modules/<module>/services/*.service.yaml` (prefix with package alias if collision) |
| workflow tags | `<module>.module.yaml` workflows |
| `define tag` / `define macro` | `registry.tags[]` / `registry.macros[]` |

Everything without `export` is **private** — consumers get `IMPORT_NOT_EXPORTED` if they import it.

### Registry exports (`registry`)

Separate from IR `exports` — controls which **tag and macro names** a package exposes to consumers.

| Field | Meaning |
| --- | --- |
| `registry.tags[]` | Exported `@name { }` definitions (`name`, `category`, `version`, `schema`) |
| `registry.macros[]` | Exported `#[name]` definitions (`name`, `category`, `version`) |
| `registry.reexports[]` | Forward symbols from a dependency without making consumers `import` the dependency |

```yaml
registry:
  tags:
    - name: compliance
      category: compliance
      version: 1
      schema: schemas/compliance-v1.json
  macros:
    - name: phi_screen
      category: compliance
      version: 1
  reexports:
    - from: "@pactia/audit-trail"
      tags: [audit_event]
      macros: []
```

`pactia package build` lowers `export define tag` / `export define macro` into `registry.*`. Authors may hand-edit `registry:`; build validates merged manifest.

**Consumer rule:** only names imported via `import` (or package prelude / `import * from`) enter the workspace effectiveRegistry.

---

## Import syntax

One keyword — **`import`** — for registry packages (`@scope/name`) and local files (`./path`). The compiler classifies the source by prefix. **`from` and `as`** are syntax tokens inside import statements, not kernel keywords.

Registry symbols (tags and macros) are **workspace-scoped** — see [registry.md — Workspace registry](registry.md#workspace-registry).

### Dependencies vs import (Cargo model)

| Layer | File | Role |
| --- | --- | --- |
| **Declare** | `pactia.toml` `[dependencies]` | Semver **ranges** — like `Cargo.toml` |
| **Pin** | `pactia.lock` | Exact **version + digest** — like `Cargo.lock` |
| **Import** | `.pactia` `import` | **Path only** — like TypeScript `import … from`; **no version in source** |

```bash
pactia add @pactia/kyc-compliance@^1.0   # writes pactia.toml + pactia.lock
```

```toml
# pactia.toml
[dependencies]
"@pactia/kyc-compliance" = "^1.0"
"@pactia/protocol-rest" = "^1.0"
```

```yaml
# pactia.lock (committed)
packages:
  - name: "@pactia/kyc-compliance"
    version: 1.0.3
    digest: sha256:abc123...
```

```pactia
import @pactia/kyc-compliance;   // resolves 1.0.3 from pactia.lock — not ^1.0 here
```

### Package resolution

All package coordinates — from `import @scope/name` **or** from an `@stack` tag target — use the **same resolver**. The compiler has no stack-specific fetch path; `@stack` is a kernel clause tag whose **target** names a package coordinate (bare id expands to `@pactia/<id>`).

| Code | Severity | Condition |
| --- | --- | --- |
| `PACKAGE_NOT_FOUND` | Error | Coordinate not found in registry |
| `PACKAGE_VERSION_UNSATISFIED` | Error | No release matches semver constraint in `pactia.toml` |
| `PACKAGE_LOCK_MISMATCH` | Error | `pactia.lock` digest differs from fetched tarball |
| `PACKAGE_DEPRECATED` | Warning | Resolved version marked deprecated on registry |
| `DEPENDENCY_NOT_DECLARED` | Error | `import @scope/name` or resolved `@stack` target absent from `pactia.toml` `[dependencies]` |
| `LOCK_ENTRY_MISSING` | Error | Declared in `pactia.toml` but no pin in `pactia.lock` — run `pactia build` |
| `VERSION_IN_IMPORT` | Error | Version range or pin appears in an `import` statement — forbidden |
| `VERSION_IN_TAG_BODY` | Error | Semver or version constraint in a tag body (e.g. `version:` inside `@stack { }`) — version belongs in `pactia.toml` / `pactia.lock` only |
| `COMPATIBLE_STACKS_UNSATISFIED` | Error | An imported package's `compatibleStacks` is not satisfied by the product's resolved stack-kind package |

### Product stack binding

`pactia.toml` `[stack].package` records which stack-kind package the product uses. That manifest field must agree with the `@stack` tag target on `product` (and with an optional `import` of the same coordinate).

| Code | Severity | Condition |
| --- | --- | --- |
| `STACK_BINDING_MISMATCH` | Error | `@stack` tag target, optional matching `import`, and `pactia.toml [stack].package` do not resolve to the same lock entry |

Implementer reference: [grammar-reference.md — Package resolution](grammar-reference.md#package-resolution).

### Import forms

```pactia
// Package prelude — default exported symbols
import @pactia/kyc-compliance;

// All exported symbols
import * from @pactia/kyc-compliance;

// Braced list
import { sanctions_check, sanctions_screen } from @acme/fintech-rules;

// Single symbol
import sanctions_check from @pactia/kyc-compliance;

// Package alias
import @pactia/kyc-compliance as kyc;

// Symbol alias
import sanctions_check as screen_check from @pactia/kyc-compliance;

// Local file merge
import "./packages/kyc-compliance/index.pactia";
```

| Form | Meaning |
| --- | --- |
| `import @scope/name;` | Package prelude — see [Prelude export semantics](#prelude-export-semantics) |
| `import * from @scope/name;` | Import **all** exported registry + AST symbols |
| `import symbol from @scope/name;` | Import one tag, macro, entity, enum, or service |
| `import { a, b } from @scope/name;` | Import listed symbols only |
| `import @scope/name as alias;` | Qualify: `@alias::tag`, `#[alias::macro]`, domain `alias.Type` |
| `import symbol as alias from @scope/name;` | Rename one imported symbol |
| `import "./path.pactia";` | Merge kernel AST from a local file (relative to importing file) |

Local `import` of a `.pactia` file merges **kernel AST** only. It does **not** register tag/macro names unless the file is a built package consumed via `import @scope/name`.

**No version** in any `import` form — not `^1.0`, not `1.2.0`. Version constraints live only in `pactia.toml`; resolved pins only in `pactia.lock`.

### Prelude export semantics

`import @scope/name;` (package prelude) is **not** the same as `import * from @scope/name;`. Prelude is a deterministic subset of exported symbols.

| Rule | Detail |
| --- | --- |
| **Registry symbols** | When `registry.prelude[]` is present in `pactia.package.yaml`, prelude imports exactly those tag and macro names (each must also appear in `registry.tags[]` / `registry.macros[]`). |
| **Default prelude** | When `registry.prelude` is **omitted**, prelude defaults to **all** exported registry tags and macros — not domain AST symbols. |
| **Empty prelude** | When `registry.prelude: []`, prelude imports **no** unqualified registry symbols. Consumers use `import { … } from`, `import symbol from`, or `import * from`. |
| **Domain AST** | `export @entity`, `export service`, `@rule`, `@integration`, and other IR symbols are **never** imported by prelude alone. Use `import * from` or explicit `import { Entity } from`. |
| **Zero exports** | A package with no exported symbols is valid. Prelude is a no-op; `import @scope/name;` still satisfies `DEPENDENCY_NOT_DECLARED` checks when declared in `pactia.toml`. |
| **Qualified access** | `import @scope/name as alias;` registers the alias for `@alias::tag` / `#[alias::macro]` even when prelude brings no unqualified names. |

At `pactia package build`, authors may set `registry.prelude` explicitly:

```yaml
registry:
  tags: [{ name: sanctions_check, category: compliance, version: 1 }]
  macros: [{ name: sanctions_screen, category: compliance, version: 1 }]
  prelude: [sanctions_check, sanctions_screen]
```

Consumers writing `import @acme/fintech-rules;` get `sanctions_check` and `sanctions_screen` in unqualified scope. They do **not** get exported entities unless they add `import * from` or a braced import.

### Workspace scope of `import`

| `import` location | Registry visible in |
| --- | --- |
| `product.pactia` | Entire workspace |
| `module.pactia` | That module subtree |
| `service.pactia` | That service subtree |
| Single-file program | Whole file |

Inner scopes inherit outer `import` statements. An `import` in `module.pactia` does not leak to sibling modules.

```
product.pactia
  import @pactia/protocol-rest;     ──▶ entire workspace

module commerce.pactia
  import @acme/payments;            ──▶ commerce subtree only
    service BillingService { }      ──▶ sees @acme/payments (inherits module import)
    service LedgerService { }         ──▶ sees @acme/payments

module fleet.pactia                   ──▶ sibling — no @acme/payments
  service FleetService { }            ──▶ product imports only
```

Registry visibility follows the same tree: a tag imported at module scope is available to every `service` block in that module, not to sibling modules.

### Qualified names after import

```pactia
import @pactia/kyc-compliance as kyc;

model {
  @entity Verification {
    status: kyc.KycStatus,
  }
}

@kyc::compliance {
  framework: hipaa,
  baa_required: true,
}
```

Domain types: `alias.TypeName`. Registry tags: `@alias::tag`. Macros: `#[alias::macro]`. Unqualified when the symbol is uniquely imported in scope.

`REGISTRY_COLLISION` → selective `import { only, needed } from @pkg;` or `import @pkg as alias;`.

### `@stack` vs `import` for stack-kind packages (e.g. rust-anb)

`@stack` is a **kernel clause tag** on `product` — same parser and tag validation as `@topology` or `@deploy`. Its **target** names a stack-kind package coordinate; resolution uses the [package resolver](#package-resolution) above, not a separate compiler path.

| Mechanism | Purpose | Example |
| --- | --- | --- |
| **`pactia.toml`** | Declare dependency + semver range; `pactia.lock` pins digest | `"@pactia/rust-anb" = "^1.0"` and `[stack] package = "@pactia/rust-anb"` |
| **`@stack rust-anb { }` on `product`** | Product **fact** in source — lowers to `product.stackId`; when the resolved package has `kind: stack`, merges `platformLaw` / `technologyPolicy` and registry `tags[]` / `macros[]` at stack-tier precedence | Inside `product { }` — **no version field** in the tag body |
| **`import @pactia/rust-anb;`** (optional) | Explicit dependency declaration; when present, coordinate must match `@stack` and lock — does **not** load registry beyond what the `@stack` tag already resolved | Top of `product.pactia` |

```pactia
import @pactia/protocol-rest;
// import @pactia/rust-anb;   // optional — same coordinate as @stack below

product FleetManagement {
  @stack rust-anb { }
  @topology { mode: microservices, }
}
```

```toml
# pactia.toml
[stack]
package = "@pactia/rust-anb"

[dependencies]
"@pactia/rust-anb" = "^1.0"
"@pactia/protocol-rest" = "^1.0"
```

**Three authorities, one coordinate — no precedence override.** The compiler checks that all three resolve to the **same** lock entry:

| Authority | What it records |
| --- | --- |
| `pactia.toml [stack].package` | Which stack-kind package the product binds to |
| `pactia.toml [dependencies]` | Semver range for that coordinate (must be present) |
| `@stack <id> { }` on `product` | Source fact — bare id expands to `@pactia/<id>` unless fully qualified |

Version and digest always come from `pactia.lock` — never from `@stack { }` or `import`. Mismatch across these three → `STACK_BINDING_MISMATCH`. Semver in `@stack { }` body → `VERSION_IN_TAG_BODY`.

See [platform.md](platform.md#stack-versions).

---

## Authoring packages

**Authors** publish packages to pactia.io (or a private registry). Every publishable package has a `pactia.package.yaml` manifest and one or more source files.

### Authoring patterns by `kind`

| `kind` | Primary source | Build command | Publish artifact |
| --- | --- | --- | --- |
| **`vertical`** | Pactia `.pactia` (`model`, `service`, roles, …) | `pactia package build` | Compiled IR YAML + manifest |
| **`stack`** | `yaml package/*` blocks in `index.pactia` | `pactia package build` | Merged `pactia.package.yaml` |
| **`protocol`** | `yaml package/*` + `schemas/*.json` | `pactia package build` | Manifest + JSON schemas |

Authors may also maintain hand-written `pactia.package.yaml` and raw `.yaml` files on disk. `pactia package build` accepts either layout and produces the same validated tarball.

### Package layout

```
@pactia/kyc-compliance/
  pactia.package.yaml    # manifest (generated or hand-authored)
  index.pactia           # package source
  README.md

@pactia/rust-anb/
  index.pactia           # stack source — yaml package/* blocks
  pactia.package.yaml    # output of pactia package build (reference)
  README.md

@pactia/protocol-rest/
  index.pactia           # optional — extensions metadata via yaml
  pactia.package.yaml
  schemas/rest-exposure-v1.json
  README.md
```

### Domain package (`.pactia`)

```pactia
pactia 1.0
// Package: @pactia/kyc-compliance

model {
  @enum {
    KycStatus { PENDING, VERIFIED, REJECTED, SUSPENDED }
  }

  @entity KycVerification {
    @pk { }
    id: uuid
    @index { }
    userId: uuid
    status: KycStatus
    @nullable { }
    verifiedAt: datetime
  }

  > KYC verification is required before financial actions
}

@integration {
  provider: SumsubKyc
  direction: inbound
  auth: {
    type: hmac
    header: X-Kyc-Signature
  }
  maps_to: POST /api/v1/webhooks/kyc

  > Identity verification provider webhooks
}
```

`pactia package build` compiles kernel declarations → `*.model.yaml`, `*.module.yaml` fragments, bundles with manifest `exports` flags.

Example: [examples/packages/kyc-compliance/](examples/packages/kyc-compliance/).

### Stack package (`yaml` — YAML-native)

Stack packages define **platform law** (`profile`, `platformLaw`, `technologyPolicy`). The kernel has no syntax for these sections — authors use `yaml package/<section>` blocks. See [platform.md](platform.md#stack-packages).

```pactia
pactia 1.0
// Package: @pactia/rust-anb

yaml package """
package:
  name: "@pactia/rust-anb"
  version: 1.0.0
  kind: stack
  pactiaVersion: "1.0"
  license: MIT
"""

yaml package/profile """
stack:
  language: rust
  framework: axum
  database: postgresql
architecture:
  pattern: clean-architecture
  sync: http-rest
  async: kafka
"""

yaml package/platformLaw """
errorEnvelope:
  code: string
  message: string
  requestId: string
pagination:
  style: CURSOR
  defaultLimit: 20
  maxLimit: 100
"""

yaml package/technologyPolicy """
use:
  language: [{ id: rust, version: ">=1.78" }]
  framework: [{ id: axum, version: "^0.7" }]
forbid:
  frameworks: [{ id: actix-web, reason: "Platform standard is Axum" }]
"""
```

Example: [examples/packages/rust-anb/](examples/packages/rust-anb/).

### Protocol package (manifest + schemas)

Protocol packages register `expose` block names. The manifest declares `extensions[]`; JSON schemas validate `expose` bodies at consumer compile time.

```pactia
yaml package """
package:
  name: "@pactia/protocol-rest"
  version: 1.0.0
  kind: protocol
  pactiaVersion: "1.0"
"""

yaml package/extensions """
- name: rest
  version: 1
  schema: schemas/rest-exposure-v1.json
  lowers_to: [services.endpoints]
  endpointSugar: true
"""
```

Example: [examples/packages/protocol-rest/](examples/packages/protocol-rest/).

---

## Package registry

Package authors register **new `#[macro]` and `@tag` names** in Pactia source. `pactia package build` lowers them into the publishable manifest — the same role as hand-written `macros[]` / `tags[]` / `extensions[]` in YAML.

**Consumers** import with `import @scope/name` and invoke `#[name]` / `@name { }`. They do **not** write `define macro` or `define tag` in product files.

### Authoring in `index.pactia`

```pactia
pactia 1.0
// Package: @acme/fintech-rules

define tag sanctions_check {
  category service
  kind compliance
  scope endpoint
  body {
    level: string
    provider: string @optional { }
  }
  lowers {
    product.yaml security.sanctionsChecks[]
    modules/{moduleKebab}/services/{serviceKebab}.service.yaml endpoints[].sanctions
  }
}

define macro sanctions_screen {
  category service
  kind compliance
  expands {
    @sanctions_check { level: enhanced, }
  }
}

model {
  @enum {
    ScreeningLevel { STANDARD, ENHANCED }
  }
}
```

### Build output (generated)

`pactia package build` emits or merges:

```yaml
# pactia.package.yaml (excerpt — generated + hand-authored metadata)
package:
  name: "@acme/fintech-rules"
  version: 1.0.0
  kind: vertical
  pactiaVersion: "1.0"

registry:
  tags:
    - name: sanctions_check
      version: 1
      allowedScopes: [endpoint]
      schema: schemas/sanctions_check-v1.json
      lowers_to:
        - file: product.yaml
          path: security.sanctionsChecks[]
        - file: modules/{moduleKebab}/services/{serviceKebab}.service.yaml
          path: endpoints[].sanctions
  macros:
    - name: sanctions_screen
      version: 1
      expands_to:
        - "@sanctions_check { level: enhanced, }"
```

JSON Schema for `body { }` is derived from field declarations. Authors may still hand-edit manifest fragments; build validates the merged result.

### Consumer compile

```
1. pactia.toml declares @acme/fintech-rules; pactia.lock pins digest
2. import { sanctions_check, sanctions_screen } from @acme/fintech-rules;
3. pactiac loads pinned tarball; imports listed symbols into workspace effectiveRegistry
4. #[sanctions_screen] expands using package macro table
5. @sanctions_check { } validates against schemas/sanctions_check-v1.json
6. Router writes per tag lowers_to (same as kernel tags)
```

### Registry errors

| Code | Condition |
| --- | --- |
| `TAG_UNKNOWN` | `@name` not in kernel, stack, std, or any `import` in scope chain |
| `MACRO_UNKNOWN` | `#[name]` not in workspace effectiveRegistry |
| `DEPENDENCY_NOT_DECLARED` | `import @scope/name` without `pactia.toml` entry |
| `VERSION_IN_IMPORT` | Semver appears in an `import` statement |
| `DEFINE_TAG_IN_PRODUCT` | `define tag` in consumer product |
| `DEFINE_MACRO_IN_PRODUCT` | `define macro` in consumer product |
| `REGISTRY_COLLISION` | Two imports expose the same unqualified tag/macro name |
| `REGISTRY_QUALIFIER_REQUIRED` | Ambiguous name — use `@alias::name` |

### Standard library (`@pactia/*`)

The **Pactia standard library** is a curated set of packages on pactia.io — not a single compiler builtin:

| Package | Provides |
| --- | --- |
| `@pactia/protocol-rest` | REST wire validation for kernel `@api { }` fields (`method`, `path`, …) (category `protocol`) |
| `@pactia/api-patterns` | Default `#[list]`, `#[paginated]`, `#[owner]`, … (overridable by stack) |
| `@pactia/surface-react` | `#[form]`, `#[a11y(...)]`, web layout macros inside `@surface { platform: web, … }` (package `kind: surface`) |
| `@pactia/surface-swiftui` | mobile layout macros inside `@surface { platform: ios, … }` (package `kind: surface`) |

`pactia init` adds default selective `import` lines (protocol + api-patterns). Stack profile **`allowedProtocolPackages`** declares which std packages a product may use — activation still requires explicit `import` (or the `pactia init` defaults). Registry macros from the package bound by `@stack` take precedence over std when both define the same name. See [registry.md — Workspace registry](registry.md#workspace-registry).

---

### Author workflow

```bash
pactia package init @acme/node-bff --kind stack   # scaffold layout
# edit index.pactia (kernel and/or yaml package/* blocks)
pactia package build                               # merge, validate, write artifact
pactia package check                               # CI-equivalent validation
pactia publish                                     # upload to pactia.io (authenticated)
```

| Command | Purpose |
| --- | --- |
| `pactia package init` | Create `pactia.package.yaml` stub + `index.pactia` for the given `kind` |
| `pactia package build` | Compile kernel, lower `define macro` / `define tag`, merge `yaml package/*` → publishable bundle |
| `pactia package check` | Validate without writing (same checks as registry CI) |
| `pactia publish` | Upload bundle; registry assigns digest |

Environment: `Pactia_REGISTRY_URL`, `Pactia_REGISTRY_TOKEN` (see below).

### Publishing rules

1. Passes `pactia package check` on all source files.
2. Valid `pactia.package.yaml` with semver and correct `kind`.
3. Stack packages: `profile`, `platformLaw`, `technologyPolicy` validated against stack schema.
4. Vertical packages: cannot export `platformLaw` or `technologyPolicy` (stack-closed law).
5. No secrets in source; automated scan on upload.
6. Digest pinned in consumer `pactia.lock` after first resolve.

---

## pactia.io registry (planned)

Public registry at **https://pactia.io** (product site and package browser; not yet built). The CLI resolves packages via **`https://registry.pactia.io/v1`** by default (`Pactia_REGISTRY_URL`).

### User flows

| Actor         | Action                                                                              |
| ------------- | ----------------------------------------------------------------------------------- |
| **Publisher** | `pactia package build` → upload bundle (`pactia.package.yaml` + IR or schemas) → CI validates → version published |
| **Consumer**  | `pactia add @pactia/kyc-compliance@^1.0` → `pactia.toml` + `pactia.lock`             |
| **Compiler**  | Resolves packages before compile; caches in `~/.pactia/registry/`                   |

### Package page (pactia.io)

Each package shows:

- README and version history
- Exported entities, integrations, rules
- Compatible `pactiaVersion` and stack packages
- Download count, checksum, publisher signature
- "Used in examples" links

### Upload requirements

1. Passes `pactia check` on all `.pactia` files
2. Valid `pactia.package.yaml` with semver
3. No duplicate coordinates
4. Automated scan: no secrets in source
5. Optional: verified publisher badge

### Lock file: `pactia.lock`

```yaml
lockVersion: 1
packages:
  - name: "@pactia/kyc-compliance"
    version: 1.0.3
    digest: sha256:abc123...
  - name: "@pactia/audit-trail"
    version: 1.2.0
    digest: sha256:def456...
```

Reproducible compiles for CI and AI agent sessions.

---

## CLI commands (planned)

**`pactia`** — package manager, language tooling, and registry (like `cargo` + publish):

```bash
pactia add @pactia/kyc-compliance@^1.0   # declare dependency, update lock
pactia build                               # resolve pins, write pactia.lock
pactia update                              # refresh dependency pins
pactia update --stack                      # refresh stack pin only
pactia init                                # new product workspace
pactia package init @scope/name --kind vertical   # scaffold package
pactia package build                       # build publishable package bundle
pactia package check                       # validate package without publishing
pactia publish                             # publish to pactia.io (authenticated)
pactia registry search kyc                 # search pactia.io
pactiac compile main.pactia -o ./input     # resolve packages then compile product
```

Environment:

| Variable                | Purpose                                 |
| ----------------------- | --------------------------------------- |
| `Pactia_REGISTRY_URL`   | Default `https://registry.pactia.io/v1` |
| `Pactia_REGISTRY_TOKEN` | Auth for publish / private packages     |

---

## Private registries

Enterprises run private registry mirrors:

```bash
export Pactia_REGISTRY_URL=https://registry.acme.corp/pactia/v1
```

Same `import @acme/internal-billing;` syntax; pactia.io is the public default.

---

## Relationship to BSC monorepo

| Location | Role |
| --- | --- |
| `examples/packages/rust-anb/` | Reference **stack package** (`yaml package/*` authoring) |
| `examples/packages/kyc-compliance/` | Reference **vertical package** (`.pactia`) |
| `examples/packages/protocol-rest/` | Reference **protocol package** |
| pactia.io | External **package** registry (community) |

---

## Security model

- Packages are **read-only** at compile time
- Compiler sandboxes: no arbitrary code execution in `.pactia` (declarative only)
- Registry signatures verify publisher identity
- `pactia audit` (planned) flags PII patterns without retention policy

---

## See also

- [Language spec](language-spec.md) — `import` and `export` grammar
- [Compilation](compilation.md) — merge order for packages
- [Overview](overview.md)

---

## Package authoring

## Two file kinds

| File kind | Root block | `define macro` / `define tag` | Compile command |
| --- | --- | --- | --- |
| **Product** | `product Identifier { }` | **Forbidden** (`DEFINE_TAG_IN_PRODUCT`, `DEFINE_MACRO_IN_PRODUCT`) | `pactiac compile` |
| **Package** | No `product` — declarations at file root | **Allowed** | `pactia package build` |

A package file may also contain kernel declarations at file root (`model { }`, `service { }`, `yaml package/*`, prose) when `kind: vertical`. Those lower to IR fragments per manifest `exports` — same kernel rules as inside a product module.

Example package source: [../fixtures/packages/fintech-rules-index.pactia](../fixtures/packages/fintech-rules-index.pactia).

---

## Registry block headers (not kernel keywords)

Pactia reserves **nine kernel keywords** — see [language-spec.md — Kernel keywords](language-spec.md#kernel-keywords-nine-reserved-words). Product authors use seven routinely; `export`, `define tag` / `define macro`, and `yaml package/*` are package-authoring forms of the same reserved words.

`scope`, `body`, `lowers`, and `expands` are **registry block headers**. They are:

- **Not** kernel keywords — the product parser never expects them at module/service scope.
- **Only** valid inside `define tag { }` or `define macro { }` in package source.
- Lowered to **`pactia.package.yaml`** (`tags[]`, `macros[]`, `schemas/*.json`) at package build — never appear in consumer product IR as syntax.

| Header | Parent block | Purpose |
| --- | --- | --- |
| `category` | `define tag`, `define macro` | Registry category: `product`, `module`, `model`, `service`, `field` (default `service`) |
| `scope` | `define tag` | Where consumers may place `@name { }` |
| `body` | `define tag` | Field schema for the tag body → JSON Schema |
| `lowers` | `define tag` | IR file + JSON path targets when the tag is used |
| `expands` | `define macro` | Lines that replace `#[name]` at consumer compile time |

**Do not confuse** registry `body { }` with kernel `@input { DtoName }` on `@api { }`. They share a name but live in different grammars:

| Syntax | Context | Meaning |
| --- | --- | --- |
| `body { level: string }` | Inside `define tag` | Package tag field schema |
| `@input CreateVehicleRequest` | Inside `@api { }` | Request DTO reference (kernel tag) |

---

## Grammar

```
PackageFile     ::= VersionDecl PackageBody
PackageBody     ::= { PackageItem }
PackageItem     ::= DefineTagDecl | DefineMacroDecl | DataDecl | ServiceDecl
                  | ModuleDecl | YamlDecl | ProseLine

DefineTagDecl   ::= "define" "tag" Identifier "{" TagDefBody "}"
TagDefBody      ::= ScopeDecl BodyDecl LowersDecl
ScopeDecl       ::= "scope" ScopeList
ScopeList       ::= Scope { Scope }
Scope           ::= "product" | "module" | "service" | "model" | "endpoint" | "field"
BodyDecl        ::= "body" "{" TagFieldDecl+ "}"
TagFieldDecl    ::= Identifier ":" ScalarKind FieldTag*
FieldTag        ::= "@" Identifier "{" "}"          // e.g. @optional { }
LowersDecl      ::= "lowers" "{" LowersLine+ "}"
LowersLine      ::= IrFilePath JsonPath
IrFilePath      ::= path under input/ (see allowlist)
JsonPath        ::= dot/bracket path; may include {serviceKebab}

DefineMacroDecl ::= "define" "macro" Identifier "{" MacroBody "}"
MacroBody       ::= "expands" "{" ExpandLine+ "}"
ExpandLine      ::= TagLine | MacroLine | KernelLine
TagLine         ::= "@" Identifier "{" TagBodyContent "}"
MacroLine       ::= "#[" Identifier MacroArgs? "]"
```

`ScalarKind` is the same set used in `@entity { }` fields: `string`, `uuid`, `int`, `decimal`, `boolean`, `datetime`, `json`, and enum type names declared in the same package or imported packages.

---

## `define tag`

Registers a new **`@name { }`** tag consumers invoke after `import`.

```pactia
define tag sanctions_check {
  scope endpoint

  body {
    level: string
    provider: string @optional { }
  }

  lowers {
    product.yaml security.sanctionsChecks[]
    modules/{moduleKebab}/services/{serviceKebab}.service.yaml endpoints[].sanctions
  }
}
```

### `scope`

Declares where `@sanctions_check { }` may appear in a consumer product.

| Scope value | Consumer may attach `@tag` inside |
| --- | --- |
| `product` | Direct child of `product { }` |
| `module` | Inside `module { }` |
| `service` | Inside `service { }` |
| `endpoint` | Inside `@api { }`, `@grpc { }`, `@graphql { }` |
| `field` | On an `@entity { }` field line |

Multiple scopes: `scope module endpoint` (space-separated on one line).

Violation → **`TAG_SCOPE_VIOLATION`**.

### `body`

Field declarations for the tag body. Package build derives **`schemas/<tagName>-v1.json`** from these lines.

| Field syntax | Lowered to JSON Schema |
| --- | --- |
| `name: string` | `type: string` |
| `name: string @optional { }` | optional property |
| `name: boolean` | `type: boolean` |
| Enum type name | `$ref` to package enum schema when exported |

Consumers write tag bodies in **assignment form** (`key: value`):

```pactia
@sanctions_check {
  level: enhanced
  provider: "refinitiv"
}
```

Validation uses the published schema → **`TAG_BODY_INVALID`** on mismatch.

### `lowers`

One line per IR emission target. Format:

```
<ir-file-path> <json-pointer-or-path-expression>
```

| Part | Rule |
| --- | --- |
| `ir-file-path` | Path relative to workspace `input/` root, e.g. `modules/{moduleKebab}/{moduleKebab}.model.yaml` |
| `json-path` | Dot/bracket path into that file's schema; `[]` = append to array |
| Placeholders | `{serviceKebab}` — replaced with enclosing service name in kebab-case at consumer compile |

Every path must appear on the **`@pactia/schema` IR allowlist** → else **`TAG_LOWERS_INVALID`**.

At consumer compile, each use of `@sanctions_check { }` writes provenance **`PACKAGE`** to the declared paths (same router as kernel tags — [registry.md](registry.md#tags)).

### Package build output

```yaml
tags:
  - name: sanctions_check
    version: 1
    allowedScopes: [endpoint]
    schema: schemas/sanctions_check-v1.json
    lowers_to:
      - file: product.yaml
        path: security.sanctionsChecks[]
      - file: modules/{moduleKebab}/services/{serviceKebab}.service.yaml
        path: endpoints[].sanctions
```

Hand-authored `tags[]` / `extensions[]` in manifest merges with lowered output; build validates the merged result.

---

## `define macro`

Registers a **`#[name]`** pattern consumers invoke after `import`.

```pactia
define macro sanctions_screen {
  expands {
    @sanctions_check { level: enhanced, }
  }
}
```

### `expands`

Contains one or more expansion lines. Each line is either:

- A **tag line** — `@tagName { ... }`
- A **macro line** — `#[otherMacro]` or `#[otherMacro(args)]`
- A **kernel fact line** allowed inside the target scope (e.g. `@cursor { default 20 }`)

Rules:

| Rule | Detail |
| --- | --- |
| Purity | Deterministic text expansion only — no conditionals on runtime values |
| References | May reference **kernel tags** and **tags/macros registered in the same package** (or already in effectiveRegistry from dependencies) |
| Nesting | `#[macro]` inside `expands { }` resolves at consumer compile after this macro expands |
| Override | Stack > explicit `import` > `@pactia/*` std — [registry.md#macros](registry.md#macros) |

Invalid reference → **`MACRO_EXPANSION_INVALID`**.

### Package build output

```yaml
macros:
  - name: sanctions_screen
    version: 1
    expands_to:
      - "@sanctions_check { level: enhanced, }"
```

---

## Kernel `model` in package files

Package source may include normal kernel blocks alongside registry definitions:

```pactia
model {
  @enum {
    ScreeningLevel { STANDARD, ENHANCED }
  }
}
```

`pactia package build` compiles kernel blocks to **`*.model.yaml`** and **`*.module.yaml`** fragments. Whether they merge into consumer products depends on **`export` modifiers** on each symbol — same rules as [Export sections (derived)](#export-sections-derived). The build writes derived manifest export flags; authors do not hand-edit coarse `domain:` / `services:` booleans.

See [packages.md](packages.md#export-sections).

---

## Consumer compile (after `import`)

```
1. Resolve `pactia.lock` digest for each package referenced in `import` statements
2. Load tarball manifest: tags[] + macros[] (+ optional IR fragments)
3. Merge exported kernel AST per `export` modifiers (see [Export sections (derived)](#export-sections-derived))
4. Insert package tags/macros into effectiveRegistry
5. Expand define template (local product only, if any)
6. Expand #[packageMacro] using effectiveRegistry
7. Validate @packageTag { } against schemas/<name>-v1.json
8. Route lowers per tag definition → input/**/*.yaml (provenance PACKAGE)
```

Consumers **invoke** only:

```pactia
import { sanctions_screen, sanctions_check } from @acme/fintech-rules;

product MyApp {
  module payments {
    service TransferService {
      @api create_transfer {
        method: POST,
        path: "/api/v1/transfers",
        #[sanctions_screen]
        @sanctions_check { level: enhanced, provider: "refinitiv", }
      }
    }
  }
}
```

---

## Package build pipeline

Distinct from product compile — see [compilation.md](compilation.md#package-build-pipeline-pactia-package-build).

```
1. Read index.pactia (+ optional hand-authored pactia.package.yaml)
2. Lower define tag → tags[] + schemas/<name>-v1.json
3. Lower define macro → macros[]
4. Compile kernel declarations (model, service, …) → IR fragments
5. Merge yaml package/* heredocs
6. Merge hand-authored tags[] / macros[] with lowered registry
7. Validate package.kind + @pactia/schema allowlist
8. Write publishable bundle + digest
```

---

## Errors

Package resolution codes: [Package resolution](#package-resolution). Registry and tag validation:

| Code | Condition |
| --- | --- |
| `DEFINE_TAG_IN_PRODUCT` | `define tag` inside consumer `product { }` |
| `DEFINE_MACRO_IN_PRODUCT` | `define macro` inside consumer `product { }` |
| `TAG_UNKNOWN` | `@name` not in kernel or any resolved `import` package |
| `MACRO_UNKNOWN` | `#[name]` not in effectiveRegistry |
| `TAG_SCOPE_VIOLATION` | `@tag` used outside declared `scope` |
| `TAG_BODY_INVALID` | Consumer tag body fails JSON Schema validation |
| `TAG_LOWERS_INVALID` | `lowers { }` path not on IR allowlist |
| `MACRO_EXPANSION_INVALID` | `expands { }` references unknown tag or macro |
| `REGISTRY_COLLISION` | Two imported packages register the same tag/macro name |

---

## See also

- [packages.md](packages.md) — manifest, kinds, exports, pactia.io workflow
- [language-spec.md](language-spec.md#define--templates-and-package-registry) — `define` in the nine-keyword model
- [registry.md#tags](registry.md#tags) — kernel vs package tags
- [registry.md](registry.md#macros) — `#[macro]` precedence and purity
- [../fixtures/packages/fintech-rules-index.pactia](../fixtures/packages/fintech-rules-index.pactia) — reference package source

---

## Extensibility

## Short answer

| Approach                                      | Verdict                                                                                       |
| --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Arbitrary user-defined **syntax/keywords**    | **No** — breaks parser, reconciler, AI, pactia.io                                             |
| **`define` for repeated spec structure**      | **Yes** — `define template` and package `define macro` / `define tag` over the fixed kernel   |
| **Packages** (`import @scope/name`)              | **Yes** — primary modularity mechanism (already specified)                                    |
| **Compile-time macros** (expand to kernel)    | **Yes** — extension mechanism 5; repeated patterns, fully validated after expansion                          |
| **Extension blocks** from registered packages | **Yes** — e.g. `@compliance` from `@pactia/hipaa` with package schema |
| **`yaml` embed** | **Yes** — raw YAML for product IR merge or package authoring (stacks, manifests); schema-validated |

**Principle:** Users never extend the **grammar**; they extend the **library**, **named templates**, and **versioned YAML packages** that lower to the same kernel AST or validated IR.

## Why not open-ended keywords?

If anyone can add `keyword foo { ... }` with custom meaning:

| Problem                     | Impact                                                      |
| --------------------------- | ----------------------------------------------------------- |
| Parser doesn't know `foo`   | IDE, pactia.io, docs, reconciler all break                  |
| Semantics undefined         | AI invents behavior at implementation time                  |
| Non-deterministic expansion | Same source → different YAML on different compiler versions |
| Security                    | Hidden logic in macro packages bypasses review              |
| Ecosystem split             | `@acme/foo` keyword means different things per org          |

That is **not** intent-first programming — it is an unconstrained DSL per project.

Pactia optimizes for: **one kernel language every agent and tool understands**, composed from **versioned, reviewable packages**.

## The kernel stays fixed

The Pactia **kernel** has **nine keywords** — see [language-spec.md](language-spec.md):

`pactia`, `product`, `module`, `service`, `model`, `import`, `export`, `define`, `yaml`

Plus block nesting inside `model`, `service`, and `module`. Everything else is **`> prose`**, **`@tag { }`**, or **`#[macro]`**. English words such as `on`, `GET`, or `POST` after `>` in a prose line are not syntax.

New product concepts should map to [extension mechanisms](#extension-mechanism-1--packages-primary) below — not invent new top-level keywords ad hoc.

## Extension mechanism 1 — Packages (primary)

**Use case:** Reuse whole domains without copy-paste.

```pactia
import @pactia/kyc-compliance;
import { escrow_hold, custody_integration } from @pactia/escrow-custody;
```

| Property                              | Deterministic? |
| ------------------------------------- | -------------- |
| Manifest declares exports             | Yes            |
| Semver + lockfile                     | Yes            |
| Merged AST validated by `@pactia/schema` | Yes            |
| Published to pactia.io with checksum  | Yes            |

**Examples:** KYC, escrow, marketplace listings, audit trail, notification patterns.

Packages are **not** new keywords — they are **bundles of kernel constructs** with a name.

## Extension mechanism 2 — `define`

**Use case:** Project-local or package-local **repeated kernel blocks** (`define template`) and **package registry authoring** (`define macro` / `define tag`).

`define` expands at compile time to existing kernel nodes. After expansion, only kernel remains in IR.

### 2.1 Endpoint templates (named patterns)

```pactia
define template crud_list(Entity, listPath, detailPath) {
  @auth { roles: [Admin] }
  #[list] #[paginated]
  @api crud_list {
    method: GET,
    path: listPath,
  }

  @auth { roles: [Admin] }
  #[detail]
  @api crud_detail {
    method: GET,
    path: detailPath,
  }
}

service AdminService {
  #[crud_list(Vehicle, "/api/v1/admin/vehicles", "/api/v1/admin/vehicles/:id")]
}
```

After expansion → `@api { }` blocks. Reconciler runs on expanded output only.

**Rules for `define template`:**

- Templates may only contain **kernel constructs**
- Parameters are identifiers (Entity names, paths) — not arbitrary code
- Expansion is **pure** (same input → same output)
- Expansion errors cite template name + line

### 2.2 Policy templates

```pactia
define template financial_retention(Entity) {
  @policy financial_retention {
    retain: { entity: Entity, period: forever, reason: "Financial audit requirement" },
  }
}

#[financial_retention(Trade)]
#[financial_retention(Dispute)]
```

### 2.3 Package registry — `define macro` and `define tag` (package only)

**Use case:** Publish new `#[name]` and `@name { }` on pactia.io in Pactia syntax — not only YAML manifest.

```pactia
// index.pactia — @acme/fintech-rules
define tag sanctions_check {
  scope endpoint
  body { level: string }
  lowers {
    security/{moduleKebab}.yaml sanctions_checks[]
  }
}

define macro sanctions_screen {
  expands {
    @sanctions_check { level: enhanced, }
  }
}
```

`pactia package build` → `tags[]`, `macros[]`, `schemas/*.json` in the tarball. Consumers `import @acme/fintech-rules` and invoke — they never write `define macro` / `define tag` in products.

See #package-registry.

---

### `define` vs `#[macro]` vs `define macro` / `define tag` — decision table

Both expand at compile time. They solve different problems and **compose** — templates often *invoke* macros; package `define macro` / `define tag` register names for others to invoke.

| Question | Answer → use |
| --- | --- |
| Repeat the same **multi-line** kernel block (`@api { }` + tags + surfaces)? | `define template name(...) { ... }` then `#[name(args)]` |
| One-line pattern shared across **all products** on a stack? | `define macro` in stack package `index.pactia` — consumers write `#[list]` |
| Stack team changes pagination defaults ecosystem-wide? | Update `define macro list` in `@pactia/rust-anb` — not product `define template` |
| New `@name { }` for a vertical (HIPAA, sanctions)? | `export define tag` in package `index.pactia` — consumers `import` the package |
| Structured fact at use site? | `@tag { }` after `import` — not `define tag` in product |
| Needs `{ }` for its meaning? | **Tag** (`define tag` in package) — one line → **macro** (`define macro` in package) |
| Org shares CRUD template across 50 repos? | `define template` in `@acme/admin-crud` package source |
| Single endpoint, written once? | Hand-write `@api { }` — no `define template` |

| | `define template` (product) | `export define macro` / `export define tag` (package) | `#[macro]` / `@tag` (import) |
| --- | --- | --- | --- |
| **Syntax** | `define template t(...) { }` | `define macro m { expands { } }` · `define tag t { scope body lowers }` | `#[name]` · `@name { }` |
| **Defined in** | Product or package `.pactia` | Package `index.pactia` only | After `import @scope/name` |
| **Scope** | Local expansion | Published registry on pactia.io | Invoke at use sites |
| **Expands to** | Kernel blocks | `macros[]` / `tags[]` in manifest | Tags → IR |
| **Provenance** | `DEFINE` | `PACKAGE` (registry) | `PACKAGE` or `MACRO` |
| **Compile phase** | Product compile — before `#[macro]` | `pactia package build` | Product compile — registry lookup |
| **Example** | `define template fleet_list(path, Dto) { @api { ... } }` | `define macro list { expands { ... } }` in `@pactia/rust-anb` | `#[list]` inside `@api { }` after `import` |

```
Product file                          Package manifest (@pactia/rust-anb)
────────────────                      ───────────────────────────────────
define template fleet_list(...) {       - name: list
  @auth { roles: [Customer, Admin] }      expands_to: ["@cursor { ... }", "#[paginated]"]
  #[list] #[paginated]    ──invoke──▶   - name: owner
  @output Dto                            expands_to: ["@filter { ... }"]
  @api fleet_list {
    method: GET,
    path: path,
  }
}
#[fleet_list(/api/v1/vehicles, VehicleListResponse)]
```

**Anti-patterns:**

| Do not | Do instead |
| --- | --- |
| `define Money = decimal` or `define X = Enum.VALUE` | Use scalar kinds (`fiatAmount: decimal`) and enum refs inline |
| `define macro list` in a consumer product | Publish `define macro list` in stack or `@pactia/api-patterns` |
| `define tag foo` in a consumer product | Publish `define tag foo` in a package on pactia.io |
| `define macro` for one endpoint | `define template` or hand-write `@api { }` |
| Bare `@list` as a tag | `#[list]` macro invocation |

See [language-spec.md](language-spec.md#define--templates-and-package-registry) and [registry.md](registry.md#macros).

## Extension mechanism 3 — Registered extension blocks

**Use case:** Vertical packs need a **registered `@tag` block** with standardized semantics — e.g. `@compliance`, `@observe`.

Prefer **`define tag`** in package `index.pactia` (mechanism 2.5). Hand-authored manifest remains valid:

```yaml
# in @pactia/hipaa pactia.package.yaml
extensions:
  - name: compliance
    schema: hipaa-compliance-v1.json
    lowers_to: [policy, rule, model.entities[].fields[].annotations]
```

Consumer writes:

```pactia
import compliance from @pactia/hipaa;

@compliance {
  phi_fields: [Patient.email, Patient.diagnosis]
  baa_required: true
  retention_clinical: 7y
}
```

| Property                | How determinism is preserved                                                               |
| ----------------------- | ------------------------------------------------------------------------------------------ |
| Block name `compliance` | Only valid if package registers it                                                         |
| Body schema             | Validated against package JSON schema before merge                                         |
| Lowering                | Fixed mapping to `policy` / `rule` / entity `annotations` — documented per package version |
| No custom runtime       | Pure compile to kernel IR                                                                  |

This feels like a "new keyword" to the author but is **deterministic** because semantics are **versioned in the package manifest**, not invented per project.

## Extension mechanism 4 — `yaml` embed

**Use case:** Stack packages and YAML-native libs; product IR fragments the kernel does not model yet.

Two modes — see [language-spec.md](language-spec.md):

| Mode | Syntax | Validated by |
| --- | --- | --- |
| Product merge | `yaml merge services/foo.yaml """..."""` | `@pactia/schema` for target IR file |
| Package authoring | `yaml package/profile """..."""` | Stack / protocol package schema at `pactia package build` |

Stack authors publish `@pactia/rust-anb` by writing `yaml package/*` blocks — not kernel syntax. Domain authors still prefer kernel `.pactia`; `yaml` is the escape hatch for YAML-shaped exports.

**Allowed:** schema-validated embeds with provenance `YAML_EMBED`.  
**Not allowed:** unvalidated arbitrary YAML in products (bypasses schema).

## Extension mechanism 5 — `#[macro]`

**Use case:** Stack- and package-owned patterns — pagination, ownership filters, rate limits — that expand to tags before IR emit.

```pactia
import @pactia/protocol-rest;
// #[list] / #[paginated] from import @pactia/api-patterns or registry precedence
// #[database] / #[cache] from package bound by @stack rust-anb { } on product

#[database]
#[cache]
#[events]
service OrderService {
  @auth { roles: [Customer] }
  #[list]
  #[paginated]
  #[owner]
  @output OrderListResponse
  @api list_orders {
    method: GET,
    path: "/api/v1/orders",
  },
  }
}
```

Macros register in stack or vertical packages — see [registry.md](registry.md#macros). Expansion is **pure** (same input → same output); conformance checks the expanded form.

**Not allowed:** macros that call external APIs, read files, or use conditionals on runtime values.

## What users should use when

| Need                                   | Use                                                 |
| -------------------------------------- | --------------------------------------------------- |
| Reuse KYC / escrow domain              | `import @pactia/...;`                           |
| Repeat list/detail on one line | `#[list]` `#[paginated]` from stack package |
| Repeat multi-line CRUD / surface blocks | `define template crud_list(...)` |
| HIPAA / SOC2 vertical block            | `@compliance` block from `@pactia/hipaa`         |
| New top-level keyword | **RFC required** — strongly discouraged; prefer `@tag` |
| One-off hack                           | **Don't** — fix kernel or package                   |

## Kernel evolution vs user keywords

| Change type                         | Process                                           |
| ----------------------------------- | ------------------------------------------------- |
| New kernel keyword                  | Pactia RFC, spec update, major version bump — **strongly discouraged** |
| New `@tag` in core registry         | Pactia RFC + [registry.md](registry.md#tags) update |
| New package on pactia.io            | Publish manifest + schema + lowering docs         |
| New `define template` in project    | Local only; must expand to existing kernel        |
| New extension block                 | Package registers in manifest; compiler whitelist |

**Users do not fork the language.** They fork **packages**.

## Determinism checklist (required for any extension)

Every extension mechanism must satisfy:

1. **Expand before validate** — `define` and `#[macro]` lower to kernel, then Zod + reconciler run
2. **Versioned** — package or macro has semver; lockfile pins digest
3. **Documented lowering** — every extension block maps to explicit IR fields
4. **No Turing completeness** — no loops, no recursion, no eval
5. **Provenance** — IR tags `PACKAGE`, `MACRO`, `DEFINE`, not `Pactia` kernel line
6. **AI-readable** — expanded spec in `specification/` includes footnote or appendix listing expansions

## Real use cases (yes, there is demand)

| Use case                               | Mechanism                                                                |
| -------------------------------------- | ------------------------------------------------------------------------ |
| Fintech: escrow + KYC + audit          | Packages                                                                 |
| Healthcare: HIPAA PHI tagging          | `@compliance` block from `@pactia/hipaa`                               |
| Marketplace: standard offer/trade CRUD | `define template marketplace_offer` in `@pactia/marketplace-core`        |
| Enterprise: internal actor model       | Private package `@acme/auth-model`                                       |
| Repeated admin CRUD per entity         | `define template`                                                        |
| Org-wide SLO defaults                  | `@observe` from stack package or `@pactia/sre-baseline` |
| Vendor webhook shape                   | Package `@sumsub/webhooks` exports `integration` + `dto`                 |

## Anti-patterns

| Anti-pattern                                        | Why                       |
| --------------------------------------------------- | ------------------------- |
| `yaml merge` without schema validation              | Bypasses contract         |
| `macro` that generates arbitrary YAML               | Bypasses schema           |
| User adds `keyword blockchain { }` without registry | Unknown semantics         |
| LLM invents new block at compile time               | Non-reproducible          |
| Extension that changes stack package                 | Violates stack-closed law |

## Extensibility roadmap (planned)

| Mechanism | Status |
| --- | --- |
| `import` | Specified |
| **`define template`** | Specified — [language-spec.md](language-spec.md#define--templates-and-package-registry) |
| **`define macro` / `define tag` (package)** | Specified — #package-registry |
| Registered `@tag` blocks from packages | Specified (`define tag` or manifest `tags[]`) |
| `yaml` embed | Specified |
| Macro registry + precedence | Specified — [registry.md](registry.md#macros) |

Do **not** add free-form user keywords without a central registry (pactia.io + compiler whitelist).

## One sentence

**Pactia stays modular through versioned packages, `@tag { }` facts, and `#[macro]` patterns — not through user-invented keywords — so every program lowers to one deterministic IR every tool and AI agent understands.**

## See also

- [platform.md](platform.md#protocol-packages) — protocol packages (`@pactia/protocol-rest`, etc.)
- [packages.md](packages.md) — pactia.io registry
- [language-spec.md](language-spec.md) — kernel keywords
- BSC vision — design laws
- [authorization](language-spec.md#authorization) — roles are kernel, not extensible per project
