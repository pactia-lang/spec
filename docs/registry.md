# Pactia registry — symbol resolution

Version: **1.2**  
Status: **Specification**

Part of: [language-spec.md](language-spec.md) | [packages.md](packages.md)

This document describes **how symbols resolve** at compile time. It is **not** a tag or macro catalog — concrete names (e.g. `@entity`, `@api`, `#list`, `@@output`) live in imported packages such as `@pactia/kernel`.

---

## Symbols

| Form         | Registered by                  | Invoked as                                                      |
| ------------ | ------------------------------ | --------------------------------------------------------------- |
| Host tag     | `export def @name in … { }`    | `@name { }` or `@name Shorthand` when `modifier,` is in the def |
| Modifier tag | `export def @@name in … { }`   | `@@name` / `@@name(Shorthand)` on next host or field            |
| Macro        | `export def #name(…) in … { }` | `#name` / `#name(args)`                                         |

Validation uses the **field spec** parsed from each `def` body (required fields, defaults, open extensions). **Every tag name uses the same validation path** — placement + field spec only. There is no separate JSON Schema sidecar per package and no tag-name-specific validation in the compiler.

---

## Tiers

All symbols come from **packages** resolved through `pactia.toml` and `pactia.lock`. There is no bundled sysroot, stack tier, or always-loaded kernel in the compiler.

| Tier           | Source                                                            | When symbols load                                |
| -------------- | ----------------------------------------------------------------- | ------------------------------------------------ |
| **dependency** | Package in `[dependencies]` **and** `import`ed in source          | Parsed `export def` from vendored `index.pactia` |
| **local**      | Non-exported `def @` / `def #` inside `module { }` in the product | Module scope only; not published                 |

`@pactia/kernel`, `@pactia/rust-stack`, and every other package resolve the same way — declare in `pactia.toml`, pin in `pactia.lock`, `import` in source. No symbol gets a special compiler branch.

Transitive package dependencies contribute symbols to `effectiveRegistry` when the direct dependency **explicitly `import`s them in its own `index.pactia`**. Packages listed only in a dependency's `pactia.toml [dependencies]` without a corresponding `import` in `index.pactia` do **not** contribute symbols. This enables packages to declare their own symbol dependencies while keeping the import graph explicit.

---

## structuralExports (1.3)

Topology packages export modules, services, models, and contexts for multi-team composition. These symbols live in **`structuralExports`** — a separate map on `effectiveRegistry` — not in the tag/macro tier.

| Field | Source | Consumer |
|-------|--------|----------|
| `tags` | `export def @` / `export def @@` in package `index.pactia` | `@name`, `@@name` in source |
| `macros` | `export def #` in package `index.pactia` | `#name(args)` in source |
| `constants` | `export def name = value` in package `index.pactia` | `${name}` interpolation |
| `contexts` | `export context` in package `index.pactia` | `context(name)` attach |
| `structuralExports` | `export module` / `service` / `model` / `context` in topology packages | `import { name } from @topology-pkg` → attach |

**Import routing by profile:**

| Package profile | Consumer import style | Resolved via |
|----------------|----------------------|--------------|
| Registry | `import { @api, #list, max_page } from @pkg` | `tags`, `macros`, `constants` |
| Topology | `import { commerce, OrderService } from @pkg` | `structuralExports` |
| Mixed | `import { @api, commerce } from @pkg` | Both; requires `mixed-exports = true` |

Bare `import @topology-pkg` (wildcard) is forbidden (`TOPOLOGY_WILDCARD_FORBIDDEN`). Use explicit `import { symbol } from …`.

---

## effectiveRegistry

Built once per compile after package resolution:

```
1. For each imported package: parse index.pactia export defs from vendored package directory
2. Derive IR slot per tag: **`body[]`** append in source order; modifiers use `merge_into_host`
3. Merge local non-exported def @ / def # from each module { } in the assembled workspace
4. Reject duplicate unqualified names across any sources (see Name collisions)
5. Attach field specs, in placements, macro splice bodies
```

Every `@name`, `@@name`, and `#name` must resolve to an entry or the compiler emits `UNKNOWN_SYMBOL`.

---

## Name collisions

**Duplicate unqualified names are always an error** — there is no shadowing or precedence winner.

| Situation                                                   | Code                 |
| ----------------------------------------------------------- | -------------------- |
| Two imported packages export the same `@` / `@@` / `#` name | `REGISTRY_COLLISION` |
| Local `def @` / `def #` collides with an imported symbol    | `REGISTRY_COLLISION` |
| Two local defs in the same module share a name              | `REGISTRY_COLLISION` |

Rename or use partial imports so only one package supplies a given symbol. Partial imports filter which defs load from a package; they do not allow the same unqualified name from two packages.

---

## Placement

Each registry entry lists **`in`** targets (`product`, `module`, `model`, `service`, `field`). The compiler checks the **enclosing block at each use site**. Mismatch → **`PLACEMENT_VIOLATION`**.

| Def `in`           | Meaning                                                                                 |
| ------------------ | --------------------------------------------------------------------------------------- |
| `product`          | May appear only inside `product { }`                                                    |
| `service, product` | May appear inside `product { }` **or** `service { }` — not in `module` or `model` alone |
| (all five targets) | May appear in any block that accepts tags                                               |

A tag used in an allowed block is **lowered into that block's IR file** — see [compilation.md — Tag lowering](compilation.md#tag-lowering).

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

|              | Product compile                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------- |
| Input        | `product.pactia`, workspace fragments, vendored package `index.pactia` files                       |
| Output       | JSON IR under the compile output directory (default layout: `input/**/*.json`; override with `-o`) |
| `export def` | Forbidden in product; required in package `index.pactia`                                           |

Package publish ships **`pactia.toml` + `index.pactia`** only — see [packages.md](packages.md).

---

## See also

- [language-spec.md](language-spec.md)
- [macros.md](macros.md)
- [compilation.md](compilation.md)
- [packages.md](packages.md)
