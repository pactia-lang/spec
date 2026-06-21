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

Validation uses the **field spec** parsed from each `def` body (required fields, defaults, open extensions). **Every tag name uses the same validation path** — placement + field spec only. There is no separate JSON Schema sidecar per package and no tag-name-specific validation in the compiler.

---

## Tiers

All symbols come from **packages** resolved through `pactia.toml` and `pactia.lock`. There is no bundled sysroot, stack tier, or always-loaded kernel in the compiler.

| Tier | Source | When symbols load |
| --- | --- | --- |
| **dependency** | Package in `[dependencies]` **and** `import`ed in source | Parsed `export def` from vendored `index.pactia` |
| **local** | Non-exported `def @` / `def #` inside `module { }` in the product | Module scope only; not published |

`@pactia/kernel`, `@pactia/rust-anb`, and every other package resolve the same way — declare in `pactia.toml`, pin in `pactia.lock`, `import` in source. No symbol gets a special compiler branch.

Transitive package dependencies do not add symbols unless re-exported by a direct dependency.

---

## effectiveRegistry

Built once per compile after package resolution:

```
1. For each imported package: parse index.pactia export defs from vendored package directory
2. Derive generic ir slot per tag from `in` placement and modifier flag only (no tag-name routing table)
3. Merge local non-exported def @ / def # from each module { } in the assembled workspace
4. Apply precedence on name collision (see below)
5. Attach field specs, in placements, macro splice bodies
```

Every `@name`, `@@name`, and `#name` must resolve to an entry or the compiler emits `UNKNOWN_SYMBOL`.

---

## Precedence

On duplicate unqualified names:

```
explicit import  >  other imported dependency  >  local def @ / def #
```

Same rules for every package — including `@pactia/kernel` and `@pactia/rust-anb`. No manifest `kind` and no reserved registry tier.

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

Errors: `DEPENDENCY_NOT_DECLARED`, `VERSION_IN_IMPORT`, `REGISTRY_COLLISION`, `DEF_IN_PRODUCT`, `IMPORT_UNUSED`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`.

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

Package publish ships **`pactia.toml` + `index.pactia`** only — see [packages.md](packages.md).

---

## See also

- [language-spec.md](language-spec.md)
- [macros.md](macros.md)
- [compilation.md](compilation.md)
- [packages.md](packages.md)
