# Pactia Compilation Guide

Status: **Specification** — Pactia 1.2 compiler pipeline.

Part of: [language-spec.md](language-spec.md) | [overview.md](overview.md#philosophy)

Pactia compiles to **AI-neutral JSON IR** (`input/**/*.json`) — not vendor-specific prompts. **BSC** renders that IR for Cursor, Claude Code, Copilot, or custom agents, and may optionally use an LLM to expand agent briefs — grounded in JSON IR, provenance `GENERATED`, never overriding formal IR facts.

---

## Pipeline overview

```
┌──────────────┐     ┌──────────────────────────────┐     ┌──────────────────────────────┐
│  *.pactia    │────▶│  input/                      │────▶│  bsc render + expand (LLM)   │
│  pactiac     │     │  *.module.json *.model.json  │     │  (optional agent briefs)     │
│              │     │  *.service.json              │     │                              │
└──────────────┘     └──────────────────────────────┘     └──────────────────────────────┘
```

`pactiac` never calls an LLM. `pactiac compile` **reads** `pactia.lock`; it does not write it. Lock updates belong to `pactia add` / `pactia build`.

---

## Compile phases

```
0.  Assemble workspace: resolve partial imports + attach (module(name) { service(…) { model(…) } }),
    or legacy merge of modules/*/module.pactia + services, into one product AST
1.  Read and validate version declaration (pactia 1.0)
2.  Lex: strip comments (never in IR)
3.  Resolve packages via pactia.toml / pactia.lock (TOML); build effectiveRegistry from
    every imported dependency's index.pactia export defs (IR slots derived at compile)
4.  Merge declarations (local fragments → imported fragments → entry file)
5.  Expand #macro until fixed point — splice def # bodies in-place (package + local);
    check in against enclosing block at each invocation
6.  Validate @tag { } bodies against def field specs (required fields; open extensions)
7.  Cross-check tag bodies, state graphs (see below)
8.  Lower @tags → JSON IR with provenance (generic slot rules below)
9.  Infer missing shapes on lowered IR per inference rules
10. Write IR files: `workspace.json` (full bundle), `manifest.json`, slice files (emitted last)
11. (optional) bsc render / expand from IR
```

See [language-spec.md — Workspace layout](language-spec.md#workspace-layout) for multi-file merge order.

### State graph validation

Phase 7: when `@states` / `@transition` are in the effective registry, validate each module's state graphs — enum membership, duplicate edges, binding to `@entity` fields, and `@transition` edges declared on `@api` hosts.

### Inference

Phase 9: after lowering, deterministic rules fill documented IR gaps. Explicit author tags always win. Inference reads **lowered IR**, not macro decoration metadata.

---

## Tag lowering

Lowering uses **enclosing scope** (which IR file) and **generic merge rules** from each tag's `in` placement and modifier flag. The compiler does **not** map tag names (e.g. `@api`, `@entity`) to fixed JSON keys — those names live in packages only.

### Scope — which IR file

| Use site (enclosing block) | IR file |
| --- | --- |
| `product { }` | `product.json` |
| `module { }` | `modules/<m>/<m>.module.json` |
| `model { }` | `modules/<m>/<m>.model.json` |
| `service { }` | `modules/<m>/services/<s>.service.json` |
| field line in `model { }` | same model file — under the owning field |

The compiler rejects tags whose `in` placement does not include the enclosing block (`PLACEMENT_VIOLATION`) before lower.

### Slot — generic paths (no tag-name table)

| Tag kind | `ir.file` | `ir.path` | `ir.merge` |
| --- | --- | --- | --- |
| Host tag (`@name { }`) | from `in` | `extensions[]` | `append_host` |
| Modifier (`@@name`) | from `in` | `modifiers` | `merge_into_host` |
| Field modifier on model line | model | `fields[]` | `field_annotation` |

Each appended host object carries the tag body fields (and `id` / `name` when the tag declares a host id). There is **no** compiler table that routes `@api` → `endpoints[]` or `@entity` → `entities[]`.

| `ir.merge` | Meaning |
| --- | --- |
| `append_host` | Tag block becomes one object pushed at `path` |
| `merge_into_host` | Modifier merges into the pending host (e.g. `@@output` before `@api`) |
| `merge_fields` | Body fields merge as a nested object at `path` |
| `field_annotation` | Field-level tag merges under the model field |

Modifier shorthand (`@@output VehicleListResponse` before `@api`) uses **`merge_into_host`** — see [language-spec.md — Tags](language-spec.md#tags).

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

Example output tree:

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
2. Resolve package imports (lockfile pins)
3. Assemble product AST (attach or legacy folder merge)
4. Continue compile phases 1–11 (parse through emit)

---

## Provenance model

| Source | Meaning |
| --- | --- |
| `Pactia` | Written directly by the author |
| `INFERRED` | Derived by a documented deterministic rule |
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
pactiac compile product.pactia -o ./out
pactiac compile -w ./my-product -o ./out
pactia build                    # vendor lock + compile
```

---

## See also

- [language-spec.md](language-spec.md)
- [registry.md](registry.md)
- [packages.md](packages.md)
- [overview.md](overview.md)
