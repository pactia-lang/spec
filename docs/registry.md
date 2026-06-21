# Pactia registry — symbol resolution

Version: **1.2**  
Status: **Specification**

Part of: [language-spec.md](language-spec.md) | [packages.md](packages.md)

This document describes **how symbols resolve** at compile time. It is **not** a tag or macro catalog — concrete names (e.g. `@entity`, `@api`, `#list`, `@@output`) live in imported packages such as `@pactia/kernel`.

---

## Symbols

| Form | Registered by | Invoked as |
| --- | --- | --- |
| Host tag | `export def @name in … { }` | `@name { }` |
| Modifier tag | `export def @@name in … { }` | `@@name` / `@@name(Shorthand)` on next host or field |
| Macro | `export def #name(…) in … { }` | `#name` / `#name(args)` |

Validation uses the **field spec** parsed from each `def` body (required fields, defaults, open extensions). Protocol packages may additionally ship a **wire JSON Schema** (`[protocol].wire-schema` in `pactia.toml`) — see [packages.md](packages.md). There is no separate JSON Schema file per kernel tag.

---

## Tiers

All symbols come from **packages** resolved through `pactia.toml` and `pactia.lock`. There is no bundled sysroot or always-loaded kernel in the compiler.

| Tier | Source | When symbols load |
| --- | --- | --- |
| **dependency** | Package in `[dependencies]` **and** `import`ed in source | Parsed `export def` from vendored `index.pactia` |
| **stack** | Stack-kind package: `import` + product-level `#stack_macro` (see [platform.md](platform.md)) | Same as dependency; wins on name collision (see precedence) |
| **local** | Non-exported `def @` / `def #` inside `module { }` in the product | Module scope only; not published |

`@pactia/kernel` is a normal package — declare it in `pactia.toml`, pin it in `pactia.lock`, and `import @pactia/kernel;` (or partial import). The compiler does not parse a special `kernel.pactia` path.

Transitive package dependencies do not add symbols unless re-exported by a direct dependency.

---

## effectiveRegistry

Built once per compile after package resolution:

```
1. For each imported package: parse index.pactia export defs from vendored package directory
2. Derive ir slot metadata per tag at compile (tag name + in placement + lowering rules)
3. Identify stack package from product-level #stack_macro; merge at stack tier
4. Merge local non-exported def @ / def # from each module { } in the assembled workspace
5. Apply precedence on name collision (see below)
6. Attach field specs, in placements, macro splice bodies
```

Every `@name`, `@@name`, and `#name` must resolve to an entry or the compiler emits `UNKNOWN_SYMBOL`.

---

## Precedence

On duplicate unqualified names:

```
stack  >  explicit import  >  other imported dependency  >  local def @ / def #
```

Stack packages may override macros (e.g. pagination defaults) when the product binds that stack.

---

## Placement

Each registry entry lists **`in`** targets (`product`, `module`, `model`, `service`, `field`). The compiler walks the enclosing block at each use site. Mismatch → **`PLACEMENT_VIOLATION`**.

**`export def` must include `in`.** Local non-exported `def @` / `def #` in `module { }` may omit `in` (all placements) — see [language-spec.md](language-spec.md).

Missing `in` on `export def` → **`DEF_PLACEMENT_REQUIRED`**.

---

## Imports and exports

- `import @scope/name;` — full import; versions live in `pactia.toml` / `pactia.lock`
- `import { @api, @@output, #list } from @scope/name;` — partial import
- `import { orders, OrderService } from ./fragments/…;` — fragment symbols (attach)
- `export def …` — package `index.pactia` only
- `export module` / `export service` / `export model` — fragment files

Errors: `DEPENDENCY_NOT_DECLARED`, `VERSION_IN_IMPORT`, `EXPORT_COLLISION`, `DEF_IN_PRODUCT`, `IMPORT_UNUSED`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`.

---

## Macro expansion

1. Find `#macro` invocations in source order
2. Resolve `def #` entry; check `in`
3. Bind parameters; substitute in **def body**
4. Splice body at invocation; recurse until fixed point
5. Validate and lower expanded `@tag` / `@@` nodes like hand-written tags

Macro **def body** may contain `@tag { }`, `@@tag`, nested `#macro`, field lines, and prose.

---

## Product compile

| | Product compile |
| --- | --- |
| Input | `product.pactia`, workspace fragments, vendored package `index.pactia` files |
| Output | `out/**/*.json` (or `input/**/*.json`) |
| `export def` | Forbidden in product; required in package `index.pactia` |

Package publish ships **`pactia.toml` + `index.pactia`** (and protocol wire schemas). No separate manifest build step — see [packages.md](packages.md).

---

## See also

- [language-spec.md](language-spec.md)
- [macros.md](macros.md)
- [compilation.md](compilation.md)
- [packages.md](packages.md)
