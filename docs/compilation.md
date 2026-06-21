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
7.  Cross-check wire fields, protocol policy, state graphs (see below)
8.  Lower @tags → JSON IR with provenance (scope + slot rules below)
9.  Infer missing shapes on lowered IR per inference rules
10. Validate output against IR JSON schemas (spec/schemas/ir/)
11. Write manifest.json (compile metadata — emitted last)
12. (optional) bsc render / expand from IR
```

See [language-spec.md — Workspace layout](language-spec.md#workspace-layout) for multi-file merge order.

### State graph validation

Phase 7: when `@states` / `@transition` are in the effective registry, validate each module's state graphs — enum membership, duplicate edges, binding to `@entity` fields, and `@transition` edges declared on `@api` hosts.

### Inference

Phase 9: after lowering, deterministic rules fill documented IR gaps. Explicit author tags always win. Inference reads **lowered IR**, not macro decoration metadata.

---

## Tag lowering

Tag bodies are not routed by author syntax in `def @`. Lowering uses **enclosing scope** (which IR file) and **registry IR metadata** (which JSON path within that file).

### Scope — which IR file

| Use site (enclosing block) | IR file |
| --- | --- |
| `product { }` | `product.json` |
| `module { }` | `modules/<m>/<m>.module.json` |
| `model { }` | `modules/<m>/<m>.model.json` |
| `service { }` | `modules/<m>/services/<s>.service.json` |
| field line in `model { }` | same model file — under the owning entity/field |

The compiler rejects tags whose `in` placement does not include the enclosing block (`PLACEMENT_VIOLATION`) before lower.

### Slot — JSON path within the file

At **product compile**, the compiler **derives** an **`ir`** object per exported tag from tag name, `in` placement, and lowering rules (`deriveIrSlotForTag`). Authors never write IR paths in Pactia source; there is no generated package manifest.

Example derived **ir** slot (internal — not authored in source):

```json
{
  "name": "api",
  "in": ["service"],
  "ir": {
    "file": "service",
    "path": "endpoints[]",
    "merge": "append_host"
  }
}
```

| `ir.merge` | Meaning |
| --- | --- |
| `append_host` | Tag opens a new host object at `path` (e.g. `@api list { }` → one endpoint) |
| `merge_into_host` | Modifier `@@` lines merge into the nearest host at `path` (e.g. `@@output`, `@@auth` on an `@api` block) |
| `merge_fields` | Tag body fields merge at `path` as a nested object |
| `field_annotation` | Field-level tag on a model line merges under that field |

Modifier shorthand (`@@output VehicleListResponse` before `@api`) merges with **`merge_into_host`** when the tag's registry entry is a modifier — see [language-spec.md — Tags](language-spec.md#tags).

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
  manifest.json
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
| `manifest.json` | Compile metadata, module index, cross-module references |
| `product.json` | Product-level lowered facts |
| `modules/<m>/<m>.module.json` | Module scope |
| `modules/<m>/<m>.model.json` | Model scope |
| `modules/<m>/services/<s>.service.json` | Service scope |

**Naming:** `<module>` kebab-case; `<service>` lowercased with optional `Service` suffix stripped (`OrderService` → `order.service.json`).

IR shape is validated by [spec/schemas/ir/](../schemas/ir/).

---

## `manifest.json`

Phase 11. Records language version, entry file, lock digest, module file tree, and cross-module `references[]` from resolved model links. Not product intent.

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

No module list in `pactia.toml`. Single-file products declare all blocks inline (see [relay.pactia](../fixtures/kernel/relay.pactia)).

### Legacy folder merge (deprecated)

1. For each `modules/<dir>/module.pactia`, merge into `product { }`.
2. Merge each `modules/<dir>/services/*.service.pactia`; resolve `import "./…"` for features under that module.

Merge order (both paths):

1. Load `pactia.toml` (stack + dependencies only)
2. Resolve package imports (lockfile pins)
3. Assemble product AST (attach or legacy folder merge)
4. Continue compile phases 1–12 (parse through manifest)

---

## Provenance model

| Source | Meaning |
| --- | --- |
| `Pactia` | Written directly by the author |
| `INFERRED` | Derived by a documented deterministic rule |
| `STACK_DEFAULT` | Supplied by the stack package |
| `PACKAGE` | Supplied by an import |
| `MACRO` | Supplied by `#macro` expansion |
| `DEFINE` | Supplied by local `def #` template expansion |
| `GUIDANCE` | Prose (`>`, `>>`) or non-enforced hints |
| `GENERATED` | Optional `bsc expand` (LLM) |
| `NOT_DERIVABLE` | IR slot exists but source does not contain the fact |

---

## Package publish

Packages ship **`pactia.toml` + `index.pactia`** (plus protocol wire schemas when `kind: protocol`). Publish is a **git tag** on the package repo — no `pactia package build` manifest step. Vendored copies load into `effectiveRegistry` at product compile. See [packages.md](packages.md).

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
