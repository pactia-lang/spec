# Pactia Compilation Guide

Status: **Specification** — Pactia 1.2 compiler pipeline.

Part of: [language-spec.md](language-spec.md) | [overview.md](overview.md#philosophy)

Pactia compiles to **AI-neutral JSON IR** — not vendor-specific prompts. Default output layout uses an `input/` directory tree; `pactiac compile -o <dir>` writes the same structure under `<dir>`. **BSC** renders that IR for Cursor, Claude Code, Copilot, or custom agents, and may optionally use an LLM to expand agent briefs — grounded in JSON IR, provenance `GENERATED`, never overriding formal IR facts.

---

## Pipeline overview

```
┌──────────────┐     ┌──────────────────────────────┐     ┌──────────────────────────────┐
│  *.pactia    │────▶│  input/  (or -o path)        │────▶│  bsc render + expand (LLM)   │
│  pactiac     │     │  *.module.json *.model.json  │     │  (optional agent briefs)     │
│              │     │  *.service.json              │     │                              │
└──────────────┘     └──────────────────────────────┘     └──────────────────────────────┘
```

`pactiac` never calls an LLM. `pactiac compile` **reads** `pactia.lock` when present; it does not write it. Lock updates belong to `pactia add` / `pactia build`.

---

## Compile phases

Canonical phase list — other docs link here rather than renumbering:

```
0.  Assemble workspace: resolve partial imports + attach (module(name) { service(…) { model(…) } }),
    or legacy merge of modules/*/module.pactia + services, into one product AST
1.  Validate version declaration (pactia 1.0)
2.  Lex: strip comments (never in IR)
3.  Parse source → AST
4.  Resolve packages via pactia.toml / pactia.lock when present; build effectiveRegistry from
    every imported dependency's index.pactia export defs (IR slots derived at compile)
5.  Bind: attach registry entries to tag/macro nodes (UNKNOWN_SYMBOL)
6.  Expand #macro until fixed point — splice def # bodies in-place (package + local);
    check in against enclosing block at each invocation
7.  Validate every tag/macro use: placement + field spec (uniform for all tag names)
8.  Lower @tags → JSON IR with provenance (generic slot rules below)
9.  Emit IR files: workspace.json (full bundle), manifest.json, slice files
— outside pactiac —
10. (optional) BSC render / expand from IR
```

See [language-spec.md — Workspace layout](language-spec.md#workspace-layout) for multi-file merge order.

**Not in pactiac today (planned / BSC):** deterministic **inference** on lowered IR (provenance `INFERRED`), cross-host consistency checks beyond field specs, tag-name-specific conformance interpretation.

### Tag validation (uniform)

Every host tag, modifier, and macro invocation is checked the same way:

1. Symbol resolves in effectiveRegistry (`UNKNOWN_SYMBOL` if not)
2. Enclosing block is listed in the symbol's `in` targets (`PLACEMENT_VIOLATION`)
3. Tag body satisfies the parsed `def` field spec — required fields present; extra fields allowed (`TAG_BODY_*`, `CLAUSE_DUPLICATE_KEY`)

There is **no** compiler pass that special-cases particular tag names (no state-graph pass, no `@api` wire pass, no kernel catalog). Cross-module or cross-host consistency beyond field specs is **out of scope for pactiac** — downstream tools (e.g. BSC conformance) may interpret lowered IR.

---

## Tag lowering

Lowering uses the **enclosing block at the use site** (which must match the symbol's `in` targets) to choose the IR file, plus **generic merge rules** from the symbol's modifier flag. The compiler does **not** map tag names (e.g. `@api`, `@entity`) to fixed JSON keys — those names live in packages only.

### Scope — which IR file

When a tag is used, the compiler maps the **enclosing block** to an IR file. The symbol's `in` must include that block; otherwise compile fails with `PLACEMENT_VIOLATION`.

| Enclosing block at use site | IR file |
| --- | --- |
| `product { }` | `product.json` |
| `module { }` | `modules/<m>/<m>.module.json` |
| `model { }` | `modules/<m>/<m>.model.json` |
| `service { }` | `modules/<m>/services/<s>.service.json` |
| field line in `model { }` | same model file — under the owning field |

Example: `export def @auth in service, product` may appear in `product { }` or `service { }`. Used in a service block → lowered to that service's `.service.json`. Used at product scope → lowered to `product.json`.

A def with `in product` only **cannot** appear inside `service { }` — placement rejects it before lower.

### Slot — generic paths (no tag-name table)

| Tag kind | `ir.path` | `ir.merge` |
| --- | --- | --- |
| Host tag (`@name { }` or `@name Shorthand`) | `extensions[]` | `append_host` |
| Modifier (`@@name`) | `modifiers` | `merge_into_host` |
| Field modifier on model line | `fields[]` | `field_annotation` |
| Local non-exported `def @` in `module { }` | per local def | `merge_fields` |

Each appended host object carries the tag body fields (and `id` / `name` when the tag declares a host id). There is **no** compiler table that routes `@api` → `endpoints[]` or `@entity` → `entities[]`.

| `ir.merge` | Meaning |
| --- | --- |
| `append_host` | Tag block becomes one object pushed at `path` |
| `merge_into_host` | Modifier merges into the pending host (e.g. `@@output` before `@api`) |
| `merge_fields` | Body fields merge as a nested object at `path` (local module defs) |
| `field_annotation` | Field-level tag merges under the model field |

Modifier shorthand (`@@output VehicleListResponse` before `@api`) uses **`merge_into_host`** — see [language-spec.md — Tags](language-spec.md#tags).

Host-tag prefix shorthand (`@auth Customer` when the package def includes `modifier,`) lowers like the equivalent block body — see [language-spec.md — Tags](language-spec.md#tags).

### Prose

Prose (`>`, `>>`) lowers with provenance **`GUIDANCE`** into the nearest applicable host's guidance array unless linked to enforceable tag fields.

---

## IR layout

IR file names mirror structural blocks: `<target>.<block>.json`.

| Pactia block | IR file |
| --- | --- |
| `module trading { }` | `trading.module.json` |
| `model { }` in module `trading` | `trading.model.json` |
| `service OrderService { }` | `order.service.json` |

Example output tree (default `-o` layout):

```text
input/
  workspace.json          # single-file bundle — start here (agents, BSC)
  manifest.json           # compile index only
  product.json
  modules/
    trading/
      trading.module.json
      trading.model.json
      services/
        order.service.json
```

| Path | Scope |
| --- | --- |
| `workspace.json` | **Full IR in one file** — `manifest`, `product`, and all module/model/service slices inline |
| `manifest.json` | Compile metadata and module file index (`pactiaVersion`, `entry`, `lockfileDigest`, `modules[]`, `references[]`) |
| `product.json` | Product-level lowered facts |
| `modules/<m>/<m>.module.json` | Module scope |
| `modules/<m>/<m>.model.json` | Model scope |
| `modules/<m>/services/<s>.service.json` | Service scope |

**Agents and tools:** read `workspace.json` first — it is the entry point referenced by kernel `@pactia`. Use per-slice files when you only need one module or service, or when diffing a single scope.

**Naming:** `<module>` kebab-case; `<service>` lowercased with optional `Service` suffix stripped (`OrderService` → `order.service.json`).

There is no JSON Schema for IR in the spec repo — only prose rules here and package `export def` field specs at compile time.

---

## `workspace.json` and `manifest.json`

**`workspace.json`** — single-file bundle of the entire compile output. Top-level keys:

| Key | Contents |
| --- | --- |
| `manifest` | Same object as `manifest.json` — compile metadata and file index |
| `product` | Same object as `product.json` — product-level intent |
| `modules` | Array of `{ module, model, services[] }` slices — same data as the per-file JSON under `modules/` |

Emitted for agents and BSC: one read gives the full product without chasing paths.

**`manifest.json`** — the `manifest` slice alone. Records language version, entry file, lock digest, module file tree (`modules[].name`, `path`, `module`, `model`, `services[].file`), and cross-module `references[]`. Not product intent — navigation metadata only.

Products with **no package dependencies** may compile without a `pactia.lock`; `lockfileDigest` in manifest is then omitted.

---

## Workspace assembly

Compile from the project root (directory containing `product.pactia`).

### Import + attach (1.2)

1. Read `product.pactia` (imports, product-level tags, attach tree).
2. Resolve `import { symbols } from ./path.pactia` — load `export module`, `export service`, `export model` bodies.
3. Splice attach references into inline `module { }` / `service { }` / `model { }` blocks.
4. Merged source contains **package imports only** (not fragment import lines).

Example attach:

```pactia
import { orders, orders_model, OrderService } from ./fragments/…;

product Relay {
  #rust_anb
  module(orders) {
    service(OrderService) {
      model(orders_model)
    }
  }
}
```

Diagnostics: `IMPORT_UNUSED`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`.

No module list in `pactia.toml`. Single-file products declare all blocks inline (see [relay.pactia](https://github.com/pactia-lang/pactiac/blob/main/test/fixtures/kernel/relay.pactia)).

### Legacy folder merge (deprecated)

1. For each `modules/<dir>/module.pactia`, merge into `product { }`.
2. Merge each `modules/<dir>/services/*.service.pactia`; resolve `import "./…"` for features under that module.

Merge order (both paths):

1. Load `pactia.toml` (dependencies only)
2. Resolve package imports (lockfile pins when lockfile present)
3. Assemble product AST (attach or legacy folder merge)
4. Continue compile phases 0–9

---

## Provenance model

| Source | Meaning |
| --- | --- |
| `Pactia` | Written directly by the author |
| `INFERRED` | Derived by a deterministic rule (**planned** — BSC or future pactiac pass; no normative inference rules in spec 1.2) |
| `PACKAGE` | Supplied by an import |
| `MACRO` | Supplied by `#macro` expansion |
| `DEFINE` | Supplied by local `def #` template expansion |
| `GUIDANCE` | Prose (`>`, `>>`) or non-enforced hints |
| `GENERATED` | Optional `bsc expand` (LLM) |
| `NOT_DERIVABLE` | IR slot exists but source does not contain the fact |

---

## Package publish

Packages ship **`pactia.toml` + `index.pactia`** only. Publish is a **git tag** on the package repo. See [packages.md](packages.md).

---

## CLI

```bash
pactiac compile product.pactia -o ./input
pactiac compile -w ./my-product -o ./input
pactia build                    # vendor lock + compile
```

`-o` sets the output root; the IR tree (`workspace.json`, `manifest.json`, slices) is written beneath it.

---

## See also

- [language-spec.md](language-spec.md)
- [registry.md](registry.md)
- [packages.md](packages.md)
- [overview.md](overview.md)
