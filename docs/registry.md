# Pactia registry — symbol resolution

Version: **1.1**  
Status: **Specification**

Part of: [language-spec.md](language-spec.md) | [packages.md](packages.md)

This document describes **how symbols resolve** at compile time. It is **not** a tag or macro catalog — concrete names (e.g. `@entity`, `@api`, `#[list]`) live in imported packages such as `@pactia/kernel`.

---

## Symbols

| Form | Registered by | Invoked as |
| --- | --- | --- |
| Tag | `export def @name in … { }` | `@name { }` or prefix shorthand when `modifier,` in def |
| Macro | `export def #name(…) in … { }` | `#[name]` / `#[name(args)]` |

Validation uses the **field spec** parsed from each `def` body (required fields, defaults, open extensions). There is no separate JSON Schema file per tag in a package tarball.

---

## Tiers

All symbols come from **packages** resolved through `pactia.toml` and `pactia.lock`. There is no bundled sysroot or always-loaded kernel in the compiler.

| Tier | Source | When symbols load |
| --- | --- | --- |
| **dependency** | Package in `[dependencies]` **and** `import`ed in source | Parsed `export def` from tarball `index.pactia` |
| **stack** | Stack-kind package bound on `product` via `@stack` / `[stack].package` | Same as dependency; wins on name collision (see precedence) |
| **local** | Non-exported `def @` / `def #` inside `module { }` in the product | Module scope only; not published |

`@pactia/kernel` is a normal package — declare it in `pactia.toml`, pin it in `pactia.lock`, and `import @pactia/kernel;` (or rely on transitive imports from other packages). The compiler does not parse a special `kernel.pactia` path.

Transitive package dependencies do not add symbols unless re-exported by a direct dependency.

---

## effectiveRegistry

Built once per compile after package resolution:

```
1. For each imported package: load pactia.package.json + parse index.pactia export defs
2. Load stack package defs when product binds a stack
3. Merge local non-exported `def @` / `def #` from each `module { }` in the assembled workspace
4. Apply precedence on name collision (see below)
5. Attach field specs, in placements, ir slot metadata, macro splice bodies
```

Every `@identifier` and `#[identifier]` must resolve to an entry or the compiler emits `UNKNOWN_SYMBOL`.

---

## Precedence

On duplicate unqualified names:

```
stack  >  explicit import  >  other imported dependency  >  local def @ / def #
```

Stack packages may override macros (e.g. pagination defaults) when the product binds that stack.

---

## Placement

Each registry entry lists **`in`** targets (`product`, `module`, `model`, `service`, `field`). The compiler walks the enclosing block at each `@tag` / `#[macro]` use site. Mismatch → **`PLACEMENT_VIOLATION`**.

**`export def` must include `in`.** Local non-exported `def @` / `def #` in `module { }` may omit `in` (all placements) — see [language-spec.md](language-spec.md).

Missing `in` on `export def` → **`DEF_PLACEMENT_REQUIRED`**.

---

## Imports and exports

- `import @scope/name;` — paths only; versions live in `pactia.toml` / `pactia.lock`
- `export def …` — only in package `index.pactia`, not in consumer `product.pactia`
- Prelude: optional manifest list of symbols auto-imported when package declares `registry.prelude[]`

Errors: `DEPENDENCY_NOT_DECLARED`, `VERSION_IN_IMPORT`, `EXPORT_COLLISION`, `DEF_IN_PRODUCT`.

---

## Macro expansion

1. Find `#[macro]` invocations in source order
2. Resolve `def #` entry; check `in`
3. Bind parameters; substitute in **def body**
4. Splice body at invocation; recurse until fixed point
5. Validate and lower expanded `@tag` nodes like hand-written tags

Macro **def body** may contain `@tag { }`, nested `#[macro]`, field lines, and prose.

---

## Product compile vs package build

| | Product compile | Package build |
| --- | --- | --- |
| Input | `product.pactia`, workspace fragments | `index.pactia` |
| Output | `input/**/*.json` | `pactia.package.json` + tarball |
| `export def` | Forbidden in product | Required for published symbols |

---

## See also

- [language-spec.md](language-spec.md)
- [macros.md](macros.md)
- [compilation.md](compilation.md)
- [packages.md](packages.md)
