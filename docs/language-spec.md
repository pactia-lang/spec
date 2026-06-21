# Pactia Language Specification

Version: **1.2**  
Status: **Specification**

Part of: [overview.md](overview.md) | [registry.md](registry.md) | [packages.md](packages.md) | [grammar-reference.md](grammar-reference.md)

Tag and macro **names** (e.g. `@api`, `#list`, `@@output`) come from **imported packages** (typically `@pactia/kernel`). This document specifies **syntax and semantics** only.

### Migration from 1.1

| 1.1 (removed) | 1.2 |
| --- | --- |
| `#[list]` | `#list` |
| `@output Type` before `@api` | `@@output Type` or `@@output(Type)` |
| `@pk` on field lines | `@@pk` |
| `modules/*/module.pactia` auto-scan | `import { … } from ./fragments/…` + `module(name) { … }` attach |
| `#[rust_anb]` inside `@stack { }` | `#rust_anb` at product level (+ `[stack].package` in `pactia.toml`) |

Legacy `#[…]` bracket macros and folder-based module discovery may still be accepted by older compilers during transition; new products should use 1.2 syntax only.

---

## Three altitudes

| Altitude | What you write | When |
| --- | --- | --- |
| **0** | `> prose` in `product { }` | Smallest legal program |
| **1** | Product prose + light `@tag` | One fact at a time |
| **2** | Full tag + macro surface | Deterministic IR, conformance |

### Altitude 0

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
import @pactia/protocol-rest;

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

Names like `@auth`, `@api`, and `@output` are package-defined symbols — not language keywords.

### Altitude 2

See [fixtures](../fixtures/kernel/relay.pactia) for a dense 1.2 example product. Legacy 1.1 altitude-2 examples (fleet) live in pactiac `test/fixtures` until refreshed to 1.2.

---

## What Pactia is

Pactia is a **shareable standard for AI-native product intent**:

- Humans read it like a product spec.
- `pactiac` lowers source to **module-scoped JSON IR** (`*.module.json`, `*.model.json`, `*.service.json`).
- BSC may render agent briefs from that IR.

---

## Design laws

1. **Small fixed keyword set** — structure, imports, `def`, prose — not a catalog of domain tags.
2. **Three sigils** — host `@tag { }`; modifier `@@tag` on the **next** `@` host or field line only; macro `#name` splices at call site.
3. **Still compilable** — tags lower deterministically; macros expand before lower.
4. **Packages on disk** — `@pactia/kernel` and other libraries are normal packages (`index.pactia`), not spec tables or compiler sysroot.
5. **Shareable** — `pactia.toml` + `pactia.lock` pin packages.
6. **No behavior scripts** — use registered outcome tags from packages when you need enforceable acceptance criteria.

---

## Keywords

### Active

| Keyword | Purpose |
| --- | --- |
| `pactia` | Version line: `pactia 1.0` |
| `product` | Root block |
| `module` | Capability group |
| `service` | Deployable API/logic unit |
| `model` | Data shapes |
| `import` | Package or file import |
| `export` | Export defs from package source only |
| `def` | Register `@tag` or `#macro` |
| `in` | Placement on `def` |

### Reserved (no syntax in 1.2)

`view`, `interface`, `class`, `function`, `field` — `field` is also an **`in`** placement target.

---

## Three sigils (line kinds)

Inside blocks, every line is exactly one of:

| Sigil | Kind | Syntax |
| --- | --- | --- |
| `@` | **Host tag** | `@identifier { … }` |
| `@@` | **Modifier tag** | `@@identifier` or `@@identifier(Shorthand)` — binds **only** to the next `@` host tag or model field line |
| `#` | **Macro** | `#identifier` or `#identifier(args)` at a statement position |
| (none) | **Prose** | `> …` or `>> … >>` |

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

| Form | Use |
| --- | --- |
| `> single line` | Agent guidance, descriptions |
| `>> multi line >>` | Delimited block; may span lines |

Interpolation: `${name}` in prose or macro bodies — compile-time only (macro parameters or module `def` constants).

Prose lowers with provenance **`GUIDANCE`** unless linked to enforceable tag fields.

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

**Modifier shorthand** — when the package `def @@` (or legacy `def @` with `modifier,`) registers a modifier, use `@@` on the line **before** the host:

```pactia
@@output VehicleListResponse
@api list_vehicles { … }

@entity Order {
  @@pk
  @@nullable
  nextCursor: string,
}
```

Block form remains valid for host tags. Modifier tags never use `{ }` at the use site unless the def specifies block args.

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

| Sigil | Def | Use |
| --- | --- | --- |
| `@` | `def @name(…)? in …? { … }` | `@name { }` |
| `@@` | `def @@name(…)? in …? { … }` | `@@name` / `@@name(Shorthand)` on next host or field |
| `#` | `def #name(…)? in …? { … }` | `#name` / `#name(args)` |

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

| Target | Invoke inside |
| --- | --- |
| `product` | `product { }` |
| `module` | `module { }` |
| `model` | `model { }` |
| `service` | `service { }` |
| `field` | field line in `model { }` |

`in service, product` — union. **Omit `in`** → all placements. **`export def` must declare `in`** in packages.

### Field spec

| Syntax | Meaning |
| --- | --- |
| `name,` | required at use site |
| `name: default,` | optional; default when omitted |
| `name: { sub, },` | nested object |
| `modifier,` | optional in `def @` only — allows prefix shorthand at use site |
| `> …` / `>> … >>` | prose in def (tags and macros) |
| `${param}` | compile-time interpolation |

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
import @pactia/protocol-rest;
```

Full import — all exports from the package. No semver in `import` — ranges in `pactia.toml`, pins in `pactia.lock` (TOML).

**Partial import** — symbols only:

```pactia
import { @api, @@output, #list, max_page } from @pactia/kernel;
import { #rust_anb, #list } from @pactia/rust-anb;
```

| List entry | Meaning |
| --- | --- |
| `@name` | Host tag def |
| `@@name` | Modifier tag def |
| `#name` | Macro def |
| `max_page` | Exported constant (no sigil) |

One style per package coordinate per file (full **or** partial, not both for the same package).

### Fragment exports and attach

Fragment files export hosts by name — no `product` block:

```pactia
export module orders { … }
export model orders_model { … }
export service OrderService { … }
export def max_page = 100
```

Import symbols, then **attach** in the product shell:

```pactia
import { orders } from ./fragments/orders.module.pactia;
import { orders_model } from ./fragments/orders.model.pactia;
import { OrderService } from ./fragments/order.service.pactia;

product Relay {
  #rust_anb

  module(orders) {
    service(OrderService) {
      model(orders_model)
    }
  }
}
```

Attach names must match imported export symbols. Diagnostics: `IMPORT_UNUSED`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`.

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

Multi-file repos compose by **import + attach** — not by scanning a `modules/` tree.

```text
my-product/
  pactia.toml
  pactia.lock
  product.pactia              # product shell: imports, attach, product-level tags
  fragments/
    orders.module.pactia      # export module orders { … }
    orders.model.pactia       # export model orders_model { … }
    order.service.pactia      # export service OrderService { … }
```

Canonical example: [relay workspace](../../pactiac/test/fixtures/workspace/relay/) (attach) and [relay.pactia](../fixtures/kernel/relay.pactia) (monolith).

### Legacy folder merge (deprecated)

Older compilers merged `modules/<dir>/module.pactia` + `services/*.service.pactia` by directory scan. New products should use **export + attach**. The compiler may still support folder merge during transition.

### Compile merge order

Workspace **assembly** (compilation phase 0):

```
1. Load pactia.toml (stack + dependencies)
2. Resolve package import paths (lockfile pins)
3. Resolve partial imports → load export module / service / model fragments
4. Splice attach tree (module(name) { service(…) { model(…) } }) into product { }
5. Emit one assembled product AST (package imports only in merged text)
```

Then run the full [compilation pipeline](compilation.md) on the assembled source (phases 1–12).

---

## Provenance

| Provenance | Meaning |
| --- | --- |
| `Pactia` | Author-written |
| `INFERRED` | Deterministic rule |
| `STACK_DEFAULT` | Stack package |
| `PACKAGE` | Import |
| `MACRO` | Macro expansion |
| `DEFINE` | Local template |
| `GUIDANCE` | Prose |
| `GENERATED` | BSC LLM expand |
| `NOT_DERIVABLE` | Slot without source fact |

---

## Compile pipeline (summary)

1. Assemble workspace
2. Parse source
3. Resolve packages → effectiveRegistry
4. Expand `#macro` until fixed point
5. Validate tag bodies against defs
6. Cross-checks (wire, states, protocol policy)
7. Lower to JSON IR
8. Infer gaps on lowered IR
9. Validate IR schemas; write `manifest.json`
10. Optional BSC

Full phase list: [compilation.md](compilation.md).

---

## Compiler alignment (1.2)

Normative spec vs **pactiac** ([feat/pactiac-1.2-compiler](https://github.com/pactia-lang/pactiac/tree/feat/pactiac-1.2-compiler)):

| Feature | Spec | pactiac |
| --- | --- | --- |
| `#macro`, `@@modifier` | Required | Supported (v2 pipeline + extract path) |
| Import + attach workspace | Required | Supported (`attach-merge`; relay fixture) |
| Partial package imports `{ @api, #list }` | Required | Supported (parse + registry filtering) |
| `${constant}` in prose | Required | Supported (v2 pipeline; imported + module constants) |
| `export module` / fragment parse at root | Required | Supported (v2 parser; attach-merge for workspace assembly) |
| `pactia package build` | Required | Supported (`index.pactia` → `pactia.package.json` with derived `ir`) |
| Legacy `#[macro]`, `modules/*` scan | Deprecated | Accepted; `LEGACY_MACRO_SYNTAX` warning; folder scan still accepted |

Fleet fixtures in pactiac retain 1.1 syntax until refreshed. **Canonical 1.2:** [relay.pactia](../fixtures/kernel/relay.pactia). Status: [plans/1.2-status.md](../../plans/1.2-status.md).

---

## Author errors

| Code | Condition |
| --- | --- |
| `PLACEMENT_VIOLATION` | Symbol used outside its `in` targets |
| `TAG_BODY_MISSING_FIELD` | Required def field missing |
| `TAG_BODY_UNKNOWN_FIELD` | Extra field (warning) |
| `MACRO_UNKNOWN` | Unknown `#name` |
| `MACRO_ARGS_INVALID` | Bad macro arity |
| `DEF_IN_PRODUCT` | `export def` in consumer product |
| `DEF_PLACEMENT_REQUIRED` | `export def` missing `in` |
| `ATTACH_UNDEFINED` | Attach references symbol not imported |
| `ATTACH_KIND_MISMATCH` | Attach expects `export module` but symbol is `export service`, etc. |
| `IMPORT_UNUSED` | Partial import symbol never referenced |

| `UNKNOWN_SYMBOL` | Unregistered `@name` / `#name` / `@@name` |

Implementer codes: [grammar-reference.md](grammar-reference.md).

---

## Anti-patterns

- Using `export def` in a product file — publish a package instead.
- Prefix `#macro` above `@api` — use in-block `#name` invocation.
- Using `modules/*` folder scan for new products — use export + attach.
- Putting semver in `import` — use `pactia.toml`.
- Expecting the language spec to list every package tag — read the package source (e.g. `@pactia/kernel` on pactia.io).

---

## See also

- [overview.md](overview.md)
- [registry.md](registry.md)
- [packages.md](packages.md)
- [compilation.md](compilation.md)
- [grammar-reference.md](grammar-reference.md)
