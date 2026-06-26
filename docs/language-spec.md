# Pactia Language Specification

Version: **1.2** (spec) — source files declare **`pactia 1.0`** on the version line.  
Status: **Specification**

Part of: [overview.md](overview.md) | [registry.md](registry.md) | [packages.md](packages.md) | [grammar-reference.md](grammar-reference.md)

Tag and macro **names** (e.g. `@api`, `#list`, `@@output`) come from **imported packages** (typically `@pactia/kernel`). This document specifies **syntax and semantics** only.

### Migration from 1.1

| 1.1 (removed)                       | 1.2                                                                                     |
| ----------------------------------- | --------------------------------------------------------------------------------------- |
| `#[list]`                           | `#list`                                                                                 |
| `@output Type` before `@api`        | `@@output Type` or `@@output(Type)`                                                     |
| `@pk` on field lines                | `@@pk`                                                                                  |
| `modules/*/module.pactia` auto-scan only | **Import + attach** — paths in `product.pactia` define composition; folder names are convention |
| `#[rust-stack]` inside `@stack { }` | `#rust-stack` at product level after `import @pactia/rust-stack` (no `[stack]` in TOML) |
| (none)                            | `context { }` keyword — external files for agent guidance (see [Context](#context))   |

Legacy `#[…]` bracket macros may still be accepted during transition. Multi-file products use **import + attach** — see [Workspace layout](#workspace-layout).

---

## Three altitudes

| Altitude | What you write               | When                          |
| -------- | ---------------------------- | ----------------------------- |
| **0**    | `> prose` in `product { }`   | Smallest legal program        |
| **1**    | Product prose + light `@tag` | One fact at a time            |
| **2**    | Full tag + macro surface     | Deterministic IR, conformance |

### Altitude 0

At least one `>` prose line in `product { }` must describe **what the product is**. Additional `>` lines are optional agent rules.

```pactia
pactia 1.0

product MyApp {
  > A mobile app for tracking personal fitness goals.
  > Never commit secrets.
}
```

### Altitude 1

```pactia
pactia 1.0

import @pactia/kernel;

product MyApp {
  > A mobile app for tracking personal fitness goals.

  module fitness {
    service WorkoutService {
      @auth Customer
      @@output WorkoutListResponse
      @api list_workouts {
        > Customers browse their workout history.
      }
    }
  }
}
```

Names like `@auth`, `@api`, and `@@output` are package-defined symbols — not language keywords.

### Altitude 2

See [relay.pactia](https://github.com/pactia-lang/pactiac/blob/main/test/fixtures/kernel/relay.pactia) for a dense 1.2 example product.

---

## What Pactia is

Pactia is a **shareable standard for AI-native product intent**:

- Humans read it like a product spec.
- `pactiac` lowers source to **module-scoped JSON IR** (`*.module.json`, `*.model.json`, `*.service.json`).
- BSC may render agent briefs from that IR.

---

## Design laws

1. **Small fixed keyword set** — structure, imports, `def`, prose, `context` — not a catalog of domain tags.
2. **Three sigils** — host `@tag { }`; modifier `@@tag` on the **next** `@` host or field line only; macro `#name` splices at call site.
3. **Still compilable** — tags lower deterministically; macros expand before lower. **All tag names share one validation and lowering path** — field spec + `in` placement only.
4. **Packages on disk** — `@pactia/kernel` and other libraries are normal packages (`index.pactia`), not spec tables or compiler sysroot.
5. **Shareable** — `pactia.toml` + `pactia.lock` pin packages.
6. **No behavior scripts** — use registered outcome tags from packages when you need enforceable acceptance criteria.

---

## Keywords

### Active

| Keyword   | Purpose                              |
| --------- | ------------------------------------ |
| `pactia`  | Version line: `pactia 1.0`           |
| `product` | Root block                           |
| `module`  | Capability group                     |
| `service` | Deployable API/logic unit            |
| `model`   | Data shapes                          |
| `import`  | Package or file import               |
| `export`  | Export defs from package source; `export module` / `service` / `model` / `context` in fragments |
| `def`     | Register `@tag` or `#macro`; or alias a `context` block (see [Context](#context)) |
| `in`      | Placement on `def`                   |
| `context` | Attach external files (md, images, …) to product / module / service / model scope |

### Reserved (no syntax in 1.2)

`view`, `interface`, `class`, `function`, `field` — `field` is also an **`in`** placement target.

---

## Three sigils (line kinds)

Inside blocks, every line is exactly one of:

| Sigil  | Kind             | Syntax                                                                                                    |
| ------ | ---------------- | --------------------------------------------------------------------------------------------------------- |
| `@`    | **Host tag**     | `@identifier { … }`                                                                                       |
| `@@`   | **Modifier tag** | `@@identifier` or `@@identifier(Shorthand)` — binds **only** to the next `@` host tag or model field line |
| `#`    | **Macro**        | `#identifier` or `#identifier(args)` at a statement position                                              |
| (none) | **Prose**        | `> …` or `>> … >>`                                                                                        |

**Modifier binding:** stacked `@@` lines apply to the same target. `#` macros do **not** sit between `@@` and their `@` host — put `#` lines **above** `@@`, then the `@` tag:

```pactia
#list
@@output(OrderListResponse)
@api list_orders { … }
```

Invalid — `@@` does not skip `#` to reach `@api`:

```pactia
@@output(OrderListResponse)
#list
@api list_orders { … }
```

Comments: `//` and `/* */` — stripped before IR.

---

## Prose

| Form               | Use                             |
| ------------------ | ------------------------------- |
| `> single line`    | Agent guidance, descriptions    |
| `>> multi line >>` | Delimited block; may span lines |

Interpolation: `${name}` in prose or macro bodies — compile-time only (macro parameters or module `def` constants).

Prose lowers with provenance **`GUIDANCE`** unless linked to enforceable tag fields.

---

## Context

**Status: Implemented** — `context` keyword lowers in pactiac; `pactia build` writes `context.index.json` and bundles files by default.

**`context`** is a language keyword — not a tag, not a macro, not in the package registry. It attaches **external files** (markdown, images, PDFs, plain text, etc.) to a scope for agent guidance. Below the conformance line, like prose.

```pactia
context api_notes {
  path: "./context/catalog/services/catalog-admin/api-notes.md",
  > API design notes for admin create flow.
}

context design_pack {
  path: [
    "./context/catalog/wireframe.png",
    "./context/catalog/flow.md",
  ],
  > Wireframes and flow for catalog admin. >>
}

context admin_assets {
  path: "./context/catalog/services/catalog-admin/",
  > All files under this folder apply to admin APIs.
}
```

### Rules

| Rule | Detail |
| ---- | ------ |
| Name | `context <name> { }` — name required |
| Body | **`path`** (required) and **prose** (`>`, `>>`) only |
| Forbidden in body | `@tags`, `#macros`, `def`, nested blocks, any other fields |
| Placement | `product { }`, `module { }`, `service { }`, `model { }` |
| Lowered IR | `{ "name", "path", "guidance?", "provenance": "Pactia" }` in **`context[]`** on the enclosing slice — see [compilation.md — Context lowering](compilation.md#context-lowering) |

### `path` shapes

One field, three value forms — all workspace-relative from **project root**:

| Form | Example | Meaning |
| ---- | ------- | ------- |
| File | `path: "./docs/vision.md"` | Single file |
| Array | `path: [ "./a.md", "./b.png" ]` | Explicit file group |
| Directory | `path: "./pack/"` | Trailing `/` — directory group (expanded at `pactia build`; see [compilation.md — Context index](compilation.md#context-index-pactia-build)) |

No globs in v1.

### Constant alias

Inside `module { }` (or service scope where `def` constants are allowed):

```pactia
def api_context_notes = context api_notes {
  path: "./context/catalog/services/catalog-admin/api-notes.md",
}

> Use ${api_context_notes} when implementing admin APIs.
```

Lowers to a resolvable reference in guidance — not enforceable IR.

### Export and attach

Mirror `export module` / `export service` / `export model`:

```pactia
// fragments/catalog-admin.context.pactia
export context api_notes {
  path: "./context/catalog/services/catalog-admin/api-notes.md",
}
```

```pactia
import { api_notes } from ./fragments/catalog-admin.context.pactia;

product Marketplace {
  module(catalog) {
    service(CatalogAdminService) {
      model(catalog_model)
      context(api_notes)
    }
  }
}
```

`context(symbol)` is valid in product / module / service / model attach trees. Multiple `context(symbol)` lines per scope are allowed. Inline `context { }` inside an exported fragment remains valid without a separate export file.

Diagnostics: `CONTEXT_IMPORT_UNUSED`, `CONTEXT_ATTACH_UNDEFINED`, `CONTEXT_ATTACH_KIND_MISMATCH` (mirror attach codes).

`pactiac` lowers `context` blocks to **`context[]`** on the enclosing IR slice (structural — not `body[]`, not `tag`). Registry `@context` tags, if used, lower to **`body[]`** with `tag: context`. It does **not** read files, validate paths, or compute digests — see [compilation.md — Context lowering](compilation.md#context-lowering).

---

## Blocks

```pactia
pactia 1.0

product Name {
  module name {
    model { … }
    service Name { … }
  }
}
```

- **`product`** — required root for consumer programs.
- **`module`** — groups services and model for one bounded context.
- **`model`** — entities, enums, relations (via registered tags).
- **`service`** — APIs and service-scoped tags.

Package **`index.pactia`** has no `product` block — only `export def` at file root.

---

## Tags

```pactia
@tag_name {
  field: value,
  nested: { a: 1, },
}
```

**Block form** — always valid:

```pactia
@auth { roles: [Customer], }
@api list_vehicles {
  method: GET,
  path: "/api/v1/vehicles",
}
```

**Host-tag prefix shorthand** — when the package `export def @name` includes **`modifier,`** in the def body, the author may omit `{ }` and supply a single shorthand token on the same line:

```pactia
@auth Customer
@auth(Admin)
```

The package def documents how shorthand maps to fields (e.g. `@auth Customer` → `roles: [Customer]`). Block form remains valid. Shorthand without `modifier,` in the package def is a **`TAG_BODY_MISSING_FIELD`** error when required fields are absent.

**Modifier shorthand** — use `def @@name` (preferred in 1.2) or legacy `def @name` with **`modifier,`** for `@@`-style binding on the line **before** the host:

```pactia
@@output VehicleListResponse
@api list_vehicles { … }

@entity Order {
  @@pk
  @@nullable
  nextCursor: string,
}
```

Block form remains valid for host tags. Modifier tags (`@@`) never use `{ }` at the use site unless the def specifies block args.

---

## Macros

C-style **call-site substitution** — not prefix decorators above the next line.

```pactia
service OrderService {
  #paginated_defaults
  @api list_orders {
    method: GET,
    path: "/api/v1/orders",
  }
}
```

Invalid:

```pactia
#paginated_defaults
@api list_orders { … }   // prefix decorator — PLACEMENT_VIOLATION / parse error
```

See [macros.md](macros.md) — unified `def` for tags and macros; **no `expands` block**.

---

## `def` — tags and macros

Tags and macros share **one** registration form. Full rules: [macros.md](macros.md).

| Sigil | Def                          | Use                                                  |
| ----- | ---------------------------- | ---------------------------------------------------- |
| `@`   | `def @name(…)? in …? { … }`  | `@name { }`                                          |
| `@@`  | `def @@name(…)? in …? { … }` | `@@name` / `@@name(Shorthand)` on next host or field |
| `#`   | `def #name(…)? in …? { … }`  | `#name` / `#name(args)`                              |

The **`def` body** holds fields, defaults, nested shapes, prose, and — for macros — lines to splice (`@tag`, `@@tag`, `#nested`, assignments). **There is no `expands` keyword.**

### Example (from package source)

```pactia
export def @sanctions_check in service {
  level,
  provider,
  > Screen against provider list.
}

export def #sanctions_screen in service {
  @sanctions_check { level: enhanced, }
}

export def #cursor_paginated(arg1) in service, model {
  pagination: "cursor pagination",
  max_page: arg1,
  > we use cursor pagination with the specific policy
}
```

### Placement (`in`)

| Target    | Invoke inside             |
| --------- | ------------------------- |
| `product` | `product { }`             |
| `module`  | `module { }`              |
| `model`   | `model { }`               |
| `service` | `service { }`             |
| `field`   | field line in `model { }` |

`in service, product` — union of allowed enclosing blocks. A symbol may only appear where its `in` includes that block; it lowers to **that block's IR file** — see [compilation.md — Tag lowering](compilation.md#tag-lowering). **Omit `in`** on local defs → all placements. **`export def` must declare `in`** in packages.

### Field spec

| Syntax            | Meaning                                                                                                                       |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `name,`           | required at use site                                                                                                          |
| `name: default,`  | optional; default when omitted                                                                                                |
| `name: { sub, },` | nested object                                                                                                                 |
| `modifier,`       | optional in `def @` only — allows **host-tag prefix shorthand** (`@name Token`) and legacy `@@`-style prefix on the next host |
| `> …` / `>> … >>` | prose in def (tags and macros)                                                                                                |
| `${param}`        | compile-time interpolation                                                                                                    |

**Validation:** required fields only; extra fields at use site allowed (`TAG_BODY_UNKNOWN_FIELD` warning).

Non-exported **`def @`** and **`def #`** inside `module { }` — local symbols for that module only (not published). **`export def`** is forbidden in consumer products.

---

## Module constants

Inside `module { }` only:

```pactia
module commerce {
  def max_page = 100
  def hint = > Validate all inputs.

  service OrderService {
    #paginated(max_page)
    @api list { … }
  }
}
```

Rules: compile-time literals or prose only; bind once; no expressions; do not lower to IR; macro args may reference these names.

---

## Literals and fields

Tag bodies use assignment syntax: `key: value`, nested `{ }`, arrays `[ … ]`, identifiers, strings, numbers, booleans.

Field lines in `model` follow the same literal grammar plus registered field-level tags.

---

## Imports, exports, and attach

### Package imports

```pactia
import @pactia/rust-stack;
```

Full import — all exports from the package. No semver in `import` — ranges in `pactia.toml`, pins in `pactia.lock` (TOML).

**Partial import** — symbols only:

```pactia
import { @api, @@output, #list, max_page } from @pactia/kernel;
import { #rust-stack, #list } from @pactia/rust-stack;
```

| List entry | Meaning                      |
| ---------- | ---------------------------- |
| `@name`    | Host tag def                 |
| `@@name`   | Modifier tag def             |
| `#name`    | Macro def                    |
| `max_page` | Exported constant (no sigil) |

One style per package coordinate per file (full **or** partial, not both for the same package).

### Fragment exports and attach

Fragment files export hosts by name — no `product` block:

```pactia
export module orders { … }
export model orders_model { … }
export service OrderService { … }
export context api_notes { … }
export def max_page = 100
```

Import symbols, then **attach** in the product shell:

```pactia
import { orders } from ./fragments/orders.module.pactia;
import { orders_model } from ./fragments/orders.model.pactia;
import { OrderService } from ./fragments/order.service.pactia;
import { api_notes } from ./fragments/catalog-admin.context.pactia;

product Relay {
  #rust-stack

  module(orders) {
    service(OrderService) {
      model(orders_model)
      context(api_notes)
    }
  }
}
```

Attach names must match imported export symbols. Diagnostics: `IMPORT_UNUSED`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`, and context-specific `CONTEXT_*` codes (see [Context](#context)).

### Package imports vs fragment imports

Fragment files **do not** declare `import … from @pactia/…`. Tags (`@api`, `@auth`, `@test`, …), modifier tags (`@@output`), and macros (`#database`, `#create`, `#rust-stack`) are imported **once** in `product.pactia`. Local `./fragments/…` imports only register **export symbols** for attach — they do not bring package definitions into the fragment file. A package import line in a fragment is **ignored** at assembly; pactiac emits **`FRAGMENT_PACKAGE_IMPORT`** (warning).

When `pactiac` assembles the workspace:

1. **Package imports** from `product.pactia` are copied to the top of the merged program.
2. **Fragment imports** are used to load `export module` / `export service` / `export model` / `export context` bodies into a symbol registry — their `import` lines are **not** kept in the merged text.
3. **Attach** splices those bodies inline (`module(name) { service(…) { model(…) context(…) } }` → nested blocks).
4. **Bind** resolves every `@` / `@@` / `#` use in inlined fragments against the registry built from the product-level `@pactia/*` imports (packages vendored via `pactia.lock`).

So a service fragment may use `@api` and `#database` without its own package import — the merged monolith already imported them at product scope. If a tag is used in a fragment but not imported in `product.pactia`, compile fails at bind with an unknown symbol.

Example ([marketplace](https://github.com/pactia-lang/examples/tree/main/marketplace)):

```pactia
// product.pactia — package + fragment imports
import { @api, @auth, @test, #database, #create, #idempotent, @@output } from @pactia/kernel;
import { CatalogAdminService } from ./fragments/catalog-admin.service.pactia;

product Marketplace {
  module(catalog) {
    service(CatalogAdminService) { model(catalog_model) }
  }
}
```

```pactia
// fragments/catalog-admin.service.pactia — no @pactia/* import
export service CatalogAdminService {
  #database
  @auth { roles: [CatalogOperator] }
  @api create_product { method: POST, path: "/api/v1/products", }
}
```

After merge, the compiler sees one program: product imports at the top, then `service CatalogAdminService { … }` with the fragment body inlined. See [compilation.md — Workspace assembly](compilation.md#workspace-assembly).

**Monolith** — all `module { }` / `service { }` / `model { }` inline in one file; attach optional.

**Inline modules** — still valid alongside attach:

```pactia
product P {
  module(orders) { service(OrderService) { model(orders_model) } }
  module platform {
    service HealthService { @api health { method: GET, path: "/health", } }
  }
}
```

**Nested modules:** not supported. `module { }` or `module(name) { }` appears only as a direct child of `product { }`.

- `export def …` only in package `index.pactia` or fragment exports — never in consumer `product.pactia` body.

---

## Authorization (concepts)

Two layers common in products:

1. **Application roles** — who can call an API (registered tags on endpoints).
2. **Party / row scope** — which rows a caller may see (macros and tags on endpoints).

Exact field names live in imported packages — not in this spec.

---

## Multi-surface

Products may declare UI intent with registered product-scoped tags and bind them to APIs in the same file. Surface-specific tags come from surface packages (`@pactia/surface-*`), not new language keywords.

---

## Workspace layout

**Folder structure is not part of the language.** The compiler does not require `modules/`, `fragments/`, or any fixed directory tree. Assembly is **import-driven**: `product.pactia` declares `import { symbols } from ./any/path/file.pactia` and wires them with `module(name) { service(Symbol) { model(…) context(…) } }`.

| Concern | Normative mechanism |
| ------- | ------------------- |
| Where fragments live | **Import paths** in `product.pactia` (any relative path) |
| How fragments compose | **Attach tree** in `product { }` |
| Package tags/macros | **Package imports** once at product scope |
| Folder names | **Convention only** — team ergonomics, not compiler rules |

### Import + attach (normative)

```pactia
import { @api, #database } from @pactia/kernel;
import { orders } from ./domains/orders/orders.module.pactia;
import { orders_model } from ./domains/orders/orders.model.pactia;
import { OrderService } from ./domains/orders/order.service.pactia;

product Marketplace {
  module(orders) {
    service(OrderService) {
      model(orders_model)
    }
  }
}
```

Fragment files use `export module` / `export service` / `export model` / `export context`. They do **not** repeat `import … from @pactia/…` — product-level package imports apply to all inlined bodies.

Diagnostics: `IMPORT_UNUSED`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`, `CONTEXT_IMPORT_UNUSED`, `CONTEXT_ATTACH_UNDEFINED`, `CONTEXT_ATTACH_KIND_MISMATCH`.

Canonical examples: [relay workspace](https://github.com/pactia-lang/pactiac/tree/main/test/fixtures/workspace/relay) (`./fragments/…`), [PPM](https://github.com/pactia-lang/examples/tree/main/ppm) (`./modules/…` — same mechanism, different folder convention).

### Recommended folder conventions (non-normative)

Teams often use predictable layouts for navigation and tooling — the compiler does not enforce them:

```text
my-product/
  pactia.toml
  pactia.lock
  product.pactia              # package imports + attach tree
  context/                    # optional — files referenced by context { path: … }
  shared/                     # optional — export def constants
  modules/<name>/             # common: one bounded context per directory
    <name>.module.pactia
    <name>.model.pactia
    services/*.service.pactia
  fragments/                  # common: flat or shallow fragment library
    orders.module.pactia
    order.service.pactia
```

Use whichever paths fit the repo; only **import paths** and the **attach tree** matter at compile time.

### Legacy folder scan (deprecated)

When `product.pactia` has **no** attach tree, older compilers may merge `modules/<dir>/module.pactia` + `services/*.service.pactia` by directory convention. **Do not rely on this** for new products — explicit import + attach is the supported model.

### Compile merge order

Workspace **assembly** (compilation phase 0):

```
1. Load `pactia.toml` (dependencies)
2. Resolve package import paths (lockfile pins)
3. Resolve fragment imports → load export module / service / model / context bodies
4. Splice attach tree into product { }
5. Emit one assembled product AST (package imports only in merged text)
```

Then run the full [compilation pipeline](compilation.md) on the assembled source (phases 0–10).

---

## Provenance

| Provenance      | Meaning                                                 |
| --------------- | ------------------------------------------------------- |
| `Pactia`        | Author-written                                          |
| `INFERRED`      | Deterministic rule (**planned** — BSC / future pactiac) |
| `PACKAGE`       | Import                                                  |
| `MACRO`         | Macro expansion                                         |
| `DEFINE`        | Local template                                          |
| `GUIDANCE`      | Prose                                                   |
| `GENERATED`     | BSC LLM expand                                          |
| `NOT_DERIVABLE` | Slot without source fact                                |

---

## Compile pipeline (summary)

Phases **0–9** are **pactiac**; phase **10** is optional **BSC**. Full list: [compilation.md — Compile phases](compilation.md#compile-phases).

1. Assemble workspace → parse → resolve packages → bind → expand macros → validate → lower → emit IR
2. Optional BSC render / expand

**Inference** on lowered IR (provenance `INFERRED`) is **not** specified in 1.2 — planned for BSC or a future pactiac pass.

---

## Compiler alignment (1.2)

Normative spec vs **pactiac** ([feat/pactiac-1.2-compiler](https://github.com/pactia-lang/pactiac/tree/feat/pactiac-1.2-compiler)):

| Feature                                           | Spec                           | pactiac                                                             |
| ------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------- |
| `#macro`, `@@modifier`                            | Required                       | Supported (v2 pipeline + extract path)                              |
| Import + attach workspace                         | Required                       | Supported (`attach-merge`; relay fixture)                           |
| Partial package imports `{ @api, #list }`         | Required                       | Supported (parse + registry filtering)                              |
| `${constant}` in prose                            | Required                       | Supported (v2 pipeline; imported + module constants)                |
| `export module` / fragment parse at root          | Required                       | Supported (v2 parser; attach-merge for workspace assembly)          |
| Crate model (`pactia.toml` + `index.pactia` only) | Required                       | Supported (no `pactia.package.json`)                                |
| Tag body + placement validation (phase 7)         | Required                       | Partial — bind + expand wired; full validate pass in progress       |
| IR infer pass (`INFERRED`)                        | Planned (BSC / future pactiac) | Not wired                                                           |
| Legacy `#[macro]`, `modules/*` scan               | Deprecated                     | Accepted; `LEGACY_MACRO_SYNTAX` warning; folder scan still accepted |

**Canonical 1.2:** [relay.pactia](https://github.com/pactia-lang/pactiac/blob/main/test/fixtures/kernel/relay.pactia). Compiler status: [pactiac CHANGELOG](https://github.com/pactia-lang/pactiac/blob/main/CHANGELOG.md).

---

## Author errors

| Code                     | Condition                                                           |
| ------------------------ | ------------------------------------------------------------------- |
| `PLACEMENT_VIOLATION`    | Symbol used outside its `in` targets                                |
| `TAG_BODY_MISSING_FIELD` | Required def field missing                                          |
| `TAG_BODY_UNKNOWN_FIELD` | Extra field (warning)                                               |
| `MACRO_UNKNOWN`          | Unknown `#name`                                                     |
| `MACRO_ARGS_INVALID`     | Bad macro arity                                                     |
| `DEF_IN_PRODUCT`         | `export def` in consumer product                                    |
| `DEF_PLACEMENT_REQUIRED` | `export def` missing `in`                                           |
| `ATTACH_UNDEFINED`       | Attach references symbol not imported                               |
| `ATTACH_KIND_MISMATCH`   | Attach expects `export module` but symbol is `export service`, etc. |
| `IMPORT_UNUSED`          | Partial import symbol never referenced                              |
| `REGISTRY_COLLISION`     | Duplicate unqualified name from any two registry sources            |
| `UNKNOWN_SYMBOL`         | Unregistered `@name` / `#name` / `@@name`                           |

Implementer codes: [grammar-reference.md](grammar-reference.md).

---

## Anti-patterns

- Using `export def` in a product file — publish a package instead.
- Prefix `#macro` above `@api` — use in-block `#name` invocation.
- Relying on **directory scan** instead of import + attach in `product.pactia`.
- Putting semver in `import` — use `pactia.toml`.
- Expecting the language spec to list every package tag — read the package source (e.g. `@pactia/kernel` on [pactia-lang/kernel](https://github.com/pactia-lang/kernel)).

---

## See also

- [overview.md](overview.md)
- [registry.md](registry.md)
- [packages.md](packages.md)
- [compilation.md](compilation.md)
- [grammar-reference.md](grammar-reference.md)
