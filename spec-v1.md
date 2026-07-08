# Pactia Specification v1

**Version:** 1.2 (spec) — source files declare `pactia 1.0` on the version line.  
**Status:** Specification — implemented by [pactiac](https://github.com/pactia-lang/pactiac).

Pactia is an intent language for the AI era. You write what must stay true about your product. The compiler lowers it to AI-neutral JSON IR. Packages distribute intent via git repos with semver tags.

---

## Language

### Program structure

```bnf
Program        ::= VersionLine ( ImportLine | ExportLine | ProductDecl )*
VersionLine    ::= "pactia" number
```

### Keywords

Active: `pactia`, `product`, `module`, `service`, `model`, `import`, `export`, `def`, `in`, `context`.  
Reserved (not used in v1): `view`, `interface`, `class`, `function`.

### Sigils

Inside blocks, every line is exactly one of:

| Sigil | Kind | Syntax |
|---|---|---|
| `@` | Host tag | `@name { … }` or `@name Shorthand` when the def includes `modifier` |
| `@@` | Modifier tag | `@@name` or `@@name(Shorthand)` — binds to the next `@` host or field line |
| `#` | Macro | `#name` or `#name(args)` — call-site substitution |
| `>` | Prose | `> single line` or `>> multi line >>` |

### Comments

`//` line comments and `/* */` block comments. Stripped before IR emission.

---

### Product

Root block for consumer programs.

```pactia
pactia 1.0

product Name {
  > At least one prose line describing the product.
  module m { … }
  @tag { … }
  #macro
  context c { path: "./doc.md" }
}
```

### Module

Groups services and model for one bounded context. Direct child of `product`. Nested modules not supported.

```pactia
module name {
  service Name { … }
  model { … }
  @tag { … }
  #macro
  def @local_tag in service { … }
  def max_page = 100
  context c { path: "./doc.md" }
}
```

### Service

Deployable API or logic unit.

```pactia
service Name {
  #macro
  @tag { … }
  > prose
  def max_page = 100
  context c { path: "./doc.md" }
}
```

### Model

Data shapes — entities, enums, relations via package-defined tags.

```pactia
model {
  @enum Status { values: [PENDING, FULFILLED] }
  @entity Order {
    @@pk
    id: uuid,
    sku: string,
  }
}
```

Field lines: `name: type,` or `name,` (optional shorthand). Arrays: `string[]`. Field modifier tags via `@@name`.

---

## Prose

```pactia
> single line of guidance

>> multi line block of guidance
that continues until the closing marker >>

${name} interpolation in prose uses module constants or imported constants.
```

Prose lowers to `body[]` entries with `kind: "prose"` and `provenance: "GUIDANCE"`.

---

## Context

Language keyword — attaches external files for agent guidance. Appears in `product`, `module`, `service`, and `model` scopes.

```pactia
context name {
  path: "./relative/file.md",
  > Optional guidance.

  path: ["./a.md", "./b.png"],  # explicit file group
  path: "./directory/",         # trailing / expands at build time
}
```

`export context name { path: "./doc.md" }` in fragment files. Attach with `context(symbol)` in the product.

Lowers to `context[]` on the enclosing IR slice (structural — not `body[]`). pactia build writes `context.index.json` and bundles files.

---

## Tags

Package-defined symbols registered via `export def @name` in package `index.pactia`. Not language keywords.

**Block form:**

```pactia
@name {
  field: value,
  nested: { a: 1 },
  > prose in tag body
}
```

**Prefix shorthand** (when package def includes `modifier`):

```pactia
@auth { roles: [Customer] }
@auth(Admin)
```

**Host ID** — identifier between tag name and body:

```pactia
@api list_orders { method: GET }
```

Becomes `"id": "list_orders"` in IR (`"name"` in model scope).

---

## Modifier tags

`@@name` binds only to the next `@` host tag or field line.

```pactia
#list
@@output(OrderListResponse)
@api list_orders { method: GET, path: "/api/v1/orders" }
```

Stacked modifiers apply to the same target:

```pactia
@@public
@@create
@api createUser { … }
```

Lowered into the next host's `modifierTags` and `modifiers` fields.

---

## Macros

Call-site substitution. The `def #` body is spliced at the invocation site. Invocations appear **inside** the block — not as prefix decorators.

```pactia
service OrderService {
  #paginated
  @api list_orders { method: GET }
}
```

Parameterized:

```pactia
export def #cursor_paginated(arg1) in service {
  pagination: "cursor pagination",
  max_page: arg1,
}

service S {
  #cursor_paginated(100)
  @api list { … }
}
```

---

## `def` — unified tag and macro registration

```bnf
DefDecl ::= ExportOpt "def" DefSigil DefName Params? InClause? "{" DefBody* "}"
DefSigil ::= "@" | "@@" | "#"
Params ::= "(" ident-list ")"
InClause ::= "in" target ("," target)*
```

| `in` target | Allowed inside |
|---|---|
| `product` | `product { }` |
| `module` | `module { }` |
| `model` | `model { }` |
| `service` | `service { }` |
| `field` | field lines in model |

`export def` must declare `in`. Local defs may omit `in` (all placements).

Field spec in def body:

```pactia
export def @auth in service {
  roles,                    # required
  scopes: [],               # optional with default
  nested: { sub, },
  modifier,                 # enables prefix shorthand
  > prose guidance
}
```

Validation: required fields must be present. Extra fields allowed (warning). All tags use the same validation path — no tag-name-specific validation.

---

## Imports

### Package

```pactia
import @pactia/kernel;
import { @api, @@output } from @pactia/kernel;
import { #rust-stack } from @pactia/rust-stack;
```

Versions in `pactia.toml` / `pactia.lock` only.

**Import aliasing** (`as` keyword) — resolve symbol collisions without renaming packages:

```pactia
import { @api as @http_api } from @pactia/kernel;
import { @api as @rpc_api } from @pactia/grpc-api;
```

Alias must match the original symbol's sigil (`@` → `@`, `@@` → `@@`, `#` → `#`).
Sigil mismatch → `IMPORT_ALIAS_SIGIL_MISMATCH`.
Alias collision with another symbol or import → `IMPORT_ALIAS_COLLISION`.
Collision resolvable via aliasing → `IMPORT_COLLISION_RESOLVABLE` (warning).

### Fragment

```pactia
import { OrderService } from ./fragments/order.service.pactia;
```

Fragment files: `export module`, `export service`, `export model`, `export context` at root — no `product` block. No `import … from @pactia/…` in fragments.

---

## Workspace assembly

Import-driven. Folder structure is convention.

**Single file:** all blocks inline in `product.pactia`.

**Multi-file (import + attach):**

```
product.pactia:
  import { @api, #database } from @pactia/kernel;
  import { orders } from ./modules/orders/orders.module.pactia;
  import { OrderService } from ./modules/orders/order.service.pactia;
  product Relay {
    #rust-stack
    module(orders) {
      service(OrderService) { model(orders_model) }
    }
  }

modules/orders/orders.module.pactia:
  export module orders {
    @actor operators { role: Operator }
  }

modules/orders/order.service.pactia:
  export service OrderService {
    @api list_orders { method: GET, path: "/api/v1/orders" }
  }
```

Attach symbols must match imported export names. Diagnostics: `IMPORT_UNUSED`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`, `CONTEXT_*` codes.

---

## Module constants

Inside `module { }` only:

```pactia
module example {
  def max_page = 100
  def hint = > Always validate.
  service S {
    #cursor_paginated(max_page)
  }
}
```

Compile-time literals. `${name}` for interpolation.

---

## Packages

### Structure

```
@scope/name/
  pactia.toml      # [package] name, version, [dependencies]
  index.pactia     # export def files only
```

### pactia.toml

```toml
[package]
name = "@pactia/rust-stack"
version = "1.0.0"

[dependencies]
"@pactia/kernel" = "^1.0"
```

Only `[package]` and `[dependencies]`. Optional fields: `exports = "topology"` (declares topology package profile), `mixed-exports = true` (escape hatch for combined registry+topology packages). No module list, no stack binding, no kind field.

### pactia.lock

```toml
lockVersion = 1

[[package]]
name = "@pactia/kernel"
version = "1.0.0"
digest = "sha256:..."
```

Required when product has package imports. Optional for altitude-0 products.

### Commands

| Command | Role |
|---|---|
| `pactia init` | Create workspace |
| `pactia add` | Add dependency, resolve lock |
| `pactia install` | Install from lock |
| `pactia update` | Re-resolve ranges |
| `pactia build` | Install + compile |
| `pactia publish --dry-run` | Validate package |

### Coordinates

```pactia
import @pactia/kernel;          # shorthand scope
import @github.com/org/repo;     # Go-style host path
```

### Resolution

1. Read pins from `pactia.lock`
2. Load vendored package from `.pactia/packages/` (or `PACTIA_VENDOR_ROOT`)
3. Parse `index.pactia` export defs into effectiveRegistry

---

## Registry

| Form | Registered by | Invoked as |
|---|---|---|
| Host tag | `export def @name in … { }` | `@name { }` or `@name Shorthand` |
| Modifier tag | `export def @@name in … { }` | `@@name` on next host or field |
| Macro | `export def #name in … { }` | `#name` / `#name(args)` |

effectiveRegistry built once per compile. Duplicate unqualified names → `REGISTRY_COLLISION` (error, no shadowing).

---

## Compilation pipeline

Phases 0–12 are pactiac. Phases 9 and 11 are reserved for future passes.
See [compilation.md](docs/compilation.md) for the authoritative phase list.

```
0.  Assemble workspace (fragment import + attach merge)
1.  Validate version (pactia 1.0)
2.  Lex / strip comments
3.  Parse → AST
4.  Resolve packages via pactia.toml / pactia.lock
5.  Build effectiveRegistry
6.  Bind: attach registry entries to tag/macro nodes
7.  Expand #macro until fixed point
8.  Validate: placement + field spec (uniform for all tags)
9.  (Reserved: CrossCheck — cross-module validation, future)
10. Lower @tags → JSON IR with provenance
11. (Reserved: Infer — deterministic inference, future)
12. Emit IR files
```

After pactiac: optional BSC render/expand, `pactia build` context index.

---

## IR layout

```
input/
  workspace.json          # single-file bundle
  manifest.json           # compile metadata
  product.json            # product-level facts
  context.index.json      # optional — pactia build
  context/                # optional — bundled files
  modules/<name>/
    <name>.module.json
    <name>.model.json
    services/<service>.service.json
```

### body[]

Source-order stream. Each entry:

```json
{ "kind": "prose", "text": "...", "provenance": "GUIDANCE" }
{ "tag": "api", "id": "list_orders", "method": "GET", "provenance": "Pactia" }
{ "tag": "entity", "name": "Order", "fields": [...], "provenance": "Pactia" }
```

All tags use the same JSON object shape — determined by the registry def. No per-type aggregation arrays.

### Modifier lowering

```json
{
  "tag": "api", "id": "list_orders",
  "modifierTags": ["output"],
  "modifiers": { "bodyRef": "OrderListResponse" }
}
```

### Field lowering

```json
{
  "name": "id", "type": "UUID", "array": false, "optional": false,
  "modifierTags": ["pk"]
}
```

---

## Provenance

| Label | Meaning |
|---|---|
| `Pactia` | Author-written |
| `PACKAGE` | From imported package |
| `MACRO` | Macro expansion |
| `DEFINE` | Local def template expansion |
| `GUIDANCE` | Prose |

---

## Diagnostic codes

### Registry / parse / bind

`UNKNOWN_SYMBOL`, `DEF_IN_PRODUCT`, `DEF_PLACEMENT_REQUIRED`, `PLACEMENT_VIOLATION`, `REGISTRY_COLLISION`, `DEPENDENCY_NOT_DECLARED`, `VERSION_IN_IMPORT`, `MACRO_UNKNOWN`, `MACRO_ARGS_INVALID`, `MACRO_EXPANSION_CYCLE`, `MACRO_EXPANSION_INVALID`, `IMPORT_UNUSED`, `UNUSED_IMPORT`, `IMPORT_MISSING`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`, `CONTEXT_IMPORT_UNUSED`, `CONTEXT_ATTACH_UNDEFINED`, `CONTEXT_ATTACH_KIND_MISMATCH`, `CONSTANT_DEF_REQUIRED`, `CONSTANT_UNDEFINED`.

### Tag body validation

`TAG_BODY_MISSING_FIELD`, `TAG_BODY_UNKNOWN_FIELD`, `TAG_BODY_INVALID`, `CLAUSE_DUPLICATE_KEY`.

### Package resolution

`PACKAGE_NOT_FOUND`, `PACKAGE_LOCK_MISMATCH`, `LOCK_ENTRY_MISSING`, `LOCK_DIGEST_MISMATCH`, `LOCK_STALE`, `LOCK_MISSING`.

### Package import resolution (1.3)

`PACKAGE_IMPORT_UNRESOLVED`, `PACKAGE_SYMBOL_UNRESOLVED`, `PACKAGE_CIRCULAR_DEPENDENCY`, `CONSUMER_REDUNDANT_IMPORT`.

### Import aliasing (1.3)

`IMPORT_ALIAS_SIGIL_MISMATCH`, `IMPORT_ALIAS_COLLISION`, `IMPORT_COLLISION_RESOLVABLE`.

### Version / parse

`UNSUPPORTED_VERSION`, `PARSE_ERROR`.

### Context keyword

`CONTEXT_FILE_NOT_FOUND`, `CONTEXT_PATH_INVALID`, `CONTEXT_DIR_EMPTY`, `CONTEXT_TOO_MANY_FILES`, `CONTEXT_DIGEST_ERROR`.

### Workspace attach (additional)

`IMPORT_DUPLICATE`, `EXPORT_KIND_AMBIGUITY`.

### Topology packages (1.3)

`TOPOLOGY_DEF_FORBIDDEN`, `TOPOLOGY_WILDCARD_FORBIDDEN`, `TOPOLOGY_NESTED_EXPORT`, `TOPOLOGY_MULTIPLE_ROOT_EXPORTS`, `TOPOLOGY_MANIFEST_INLINE_EXPORT`, `TOPOLOGY_EXPORT_FILE_MISSING`, `PACKAGE_EXPORT_MIXED`, `PACKAGE_PROFILE_MISMATCH`, `HYBRID_PACKAGE_DISCOURAGED`, `PACKAGE_IMPORT_MIXED`, `EXPORT_NOT_DECLARED`, `TOPOLOGY_DUPLICATE_SERVICE`.

---

## Implemented packages

| Package | Location |
|---|---|
| `@pactia/kernel` | [kernel/](https://github.com/pactia-lang/kernel) — `@entity`, `@api`, `@auth`, `@enum`, `@test`, `@@output`, `@@pk`, `@actor`, `@rule`, `@topology`, `@guide`, etc. |
| `@pactia/rust-stack` | [rust-stack/](https://github.com/pactia-lang/pactia-io/tree/main/rust-stack) — `#rust-stack` |
| `@pactia/html-css-js` | [html-css-js/](https://github.com/pactia-lang/pactia-io/tree/main/html-css-js) — web stack macros |

---

## Canonical examples

| Example | Description |
|---|---|
| [relay.pactia](https://github.com/pactia-lang/pactiac/blob/main/test/fixtures/kernel/relay.pactia) | Canonical monolith (altitude 2) |
| [relay workspace](https://github.com/pactia-lang/pactiac/tree/main/test/fixtures/workspace/relay) | Multi-file import + attach |
| [marketplace](https://github.com/pactia-lang/examples/tree/main/marketplace) | Multi-module product |
| [PPM](https://github.com/pactia-lang/examples/tree/main/ppm) | Extended-tier workspace with context |

---

## Tooling

| Tool | Repo | Role |
|---|---|---|
| pactiac | [pactia-lang/pactiac](https://github.com/pactia-lang/pactiac) | Compiler |
| pactia | [pactia-lang/pactia](https://github.com/pactia-lang/pactia) | Package manager |
| vscode-pactia | [pactia-lang/vscode-pactia](https://github.com/pactia-lang/vscode-pactia) | VS Code extension |
| Examples | [pactia-lang/examples](https://github.com/pactia-lang/examples) | Canonical workspaces |

**Model-agnostic by design.** Same `.pactia` files work with Cursor, Claude Code, Copilot, or custom agents.

---

## License

MIT