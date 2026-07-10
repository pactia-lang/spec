# Pactia Compilation Guide

Status: **Specification** — Pactia 1.2 compiler pipeline.

Part of: [language-spec.md](language-spec.md) | [overview.md](overview.md#philosophy)

Pactia compiles to **AI-neutral YAML IR** (or JSON with `--json`) — not vendor-specific prompts. Default output is `.yaml` files under an `input/` directory tree; `pactiac compile -o <dir>` writes the same structure under `<dir>`. **BSC** renders that IR for Cursor, Claude Code, Copilot, or custom agents, and may optionally use an LLM to expand agent briefs — grounded in IR, provenance `GENERATED`, never overriding formal IR facts.

---

## Pipeline overview

```
┌──────────────┐     ┌──────────────────────────────┐     ┌──────────────────────────────┐
│  *.pactia    │────▶│  input/  (or -o path)        │────▶│  bsc render + expand (LLM)   │
│  pactiac     │     │  *.module.yaml *.model.yaml  │     │  (optional agent briefs)     │
│              │     │  *.service.yaml (*.json with │     │                              │
│              │     │  --json)                     │     │                              │
└──────────────┘     └──────────────────────────────┘     └──────────────────────────────┘
```

`pactiac` never calls an LLM. `pactiac compile` **reads** `pactia.lock` when present; it does not write it. Lock updates belong to `pactia add` / `pactia update`.

---

## Compile phases

Canonical phase list — other docs link here rather than renumbering:

```
0.  Assemble workspace: resolve fragment imports + attach in `product.pactia` (folder-agnostic paths) into one product AST
1.  Validate version declaration (pactia 1.0)
2.  Lex: strip comments (never in IR)
3.  Parse source → AST
4.  Resolve packages via pactia.toml / pactia.lock when present
5.  Build effectiveRegistry from every imported dependency's index.pactia export defs
    (IR slots derived at compile)
6.  Bind: attach registry entries to tag/macro nodes (UNKNOWN_SYMBOL)
7.  Expand #macro until fixed point — splice def # bodies in-place (package + local);
    check in against enclosing block at each invocation
8.  Validate every tag/macro use: placement + field spec (uniform for all tag names)
9.  CrossCheck (reserved — cross-module validation, future)
10. Lower @tags → JSON IR with provenance (source-order `body[]`); lower `context` keyword → `context[]` on slice (see [Context lowering](#context-lowering))
11. Infer (reserved — deterministic inference, future BSC / pactiac pass)
12. Emit IR files: workspace.yaml (full bundle), manifest.yaml, slice files (YAML default; `.json` with `--json`)
— outside pactiac —
13. (optional) BSC render / expand from IR
14. (optional) pactia build: context index — verify paths, expand directories, digests (see [Context index](#context-index-pactia-build))
```

See [language-spec.md — Workspace layout](language-spec.md#workspace-layout) for multi-file merge order.

**Not in pactiac today (planned / BSC):** deterministic **inference** on lowered IR (provenance `INFERRED`), cross-host consistency checks beyond field specs, tag-name-specific conformance interpretation.

### Tag validation (uniform)

Every host tag, modifier, and macro invocation is checked the same way:

1. Symbol resolves in effectiveRegistry (`UNKNOWN_SYMBOL` if not)
2. Enclosing block is listed in the symbol's `in` targets (`PLACEMENT_VIOLATION`)
3. Tag body satisfies the parsed `def` field spec — required fields present; extra fields allowed (`TAG_BODY_*`, `CLAUSE_DUPLICATE_KEY`)

There is **no** compiler pass that special-cases particular tag names for **validation** (no state-graph pass, no `@api` wire pass). Cross-module or cross-host consistency beyond field specs is **out of scope for pactiac** — downstream tools (e.g. BSC conformance) may interpret lowered IR.

**Lowering** preserves **source order**. The compiler walks the bound tree in document order. Host tags, prose lines, and macro expansions append to **`body[]`**; **`context`** blocks append to **`context[]`** on the same slice. Registry host entries carry **`tag`** plus def-shaped fields — agents read kind from `tag`, not from aggregation buckets.

---

## Tag lowering

Lowering uses the **enclosing block** to choose the **IR file**. Within that file, host tags, prose lines, and macro expansions append to **`body[]`** in source order. The **`context`** keyword appends to **`context[]`** on the same slice (structural slot — not mixed into `body[]`). Modifiers (`@@name`) do **not** get their own `body[]` slot — they merge into the **next** host tag (see [merge rules](#slot--merge-rules)). Macros expand **in place** during the expand-macros phase before lower, so spliced tags appear at the `#macro` site in `body[]`.

**Why order matters:** IR is the agent source of truth. Authors sequence prose before policy before stack before modules deliberately. `@@output` before `@api` binds to that API, not the next one. Aggregating `@entity` / `@enum` / `@api` into separate typed arrays destroys document order and breaks those semantics.

### Scope — which IR file

When a tag is used, the compiler maps the **enclosing block** to an IR file. The symbol's `in` must include that block; otherwise compile fails with `PLACEMENT_VIOLATION`.

| Enclosing block at use site | IR file                                  |
| --------------------------- | ---------------------------------------- |
| `product { }`               | `product.json`                           |
| `module { }`                | `modules/<m>/<m>.module.json`            |
| `model { }`                 | `modules/<m>/<m>.model.json`             |
| `service { }`               | `modules/<m>/services/<s>.service.json`  |
| field line in `model { }`   | same model file — under the owning field |

Example: `export def @auth in service, product` may appear in `product { }` or `service { }`. Used in a service block → lowered to that service's `.service.json`. Used at product scope → lowered to `product.json`.

A def with `in product` only **cannot** appear inside `service { }` — placement rejects it before lower.

### Source-order `body[]`

Each IR slice (`product`, `module`, `model`, `service`) has structural **`name`**, ordered **`body[]`**, and optional ordered **`context[]`**:

| IR slice root | `body[]` | `context[]` |
| ------------- | -------- | ----------- |
| `product.json` → `product` | tags, prose, macro expansions | context keyword blocks |
| `*.module.json` → `module` | same | same |
| `*.model.json` → `model` | same | same |
| `*.service.json` → `service` | same | same |

`context[]` preserves source order **among context blocks** on that slice. Relative order between a context block and a `body[]` entry is not encoded in a single stream — use both arrays on the slice.

Nested host tags (e.g. `@surface` inside `@api`) append to the parent host's **`body[]`**, preserving order within the parent.

**Structural fields** on block headers (`service.name`, `service.flags.*` from field lines and service macros) stay on the service root object — not in `body[]`.

| Source construct | IR slot |
| ---------------- | ------- |
| `>` / `>>` prose line | `{ "kind": "prose", "text": "…", "provenance": "GUIDANCE" }` in **`body[]`** |
| `context id { path: … }` | `{ "name", "path", … }` in **`context[]`** — structural keyword slot (not `body[]`, not `tag`) |
| `@context` (registry tag, if defined) | `{ "tag": "context", … }` in **`body[]`** — distinct from the keyword |
| `@guide` prose-only lines | `{ "tag": "guide", "text": "…", "provenance": "GUIDANCE" }` |
| `@entity`, `@api`, `@rule`, … | `{ "tag": "<symbol>", …fields from def…, "provenance": "Pactia" }` |
| `#macro` expansion | Spliced tags/prose appear here in macro expansion order |

**Fixed JSON keys per tag** means each entry's **object shape** is determined by the registry `def` for that `tag` (`method`/`path` for `api`, `name`/`fields` for `entity`, …) — not separate top-level buckets like `entities[]` vs `enums[]`.

Example — source order preserved:

```pactia
model {
  @enum Status { values: [PENDING, FULFILLED] }
  @entity Order { id: uuid }
  @enum Priority { values: [LOW, HIGH] }
}
```

```json
{
  "model": {
    "body": [
      { "tag": "enum", "name": "Status", "values": ["PENDING", "FULFILLED"], "provenance": "Pactia" },
      { "tag": "entity", "name": "Order", "fields": [ … ], "provenance": "Pactia" },
      { "tag": "enum", "name": "Priority", "values": ["LOW", "HIGH"], "provenance": "Pactia" }
    ]
  }
}
```

Service example — modifier binds to the following host:

```pactia
@@output OrderListResponse
@api list_orders { method: GET, path: "/api/v1/orders" }
@auth { roles: [Operator] }
@api create_order { method: POST, path: "/api/v1/orders" }
```

```json
{
  "service": {
    "body": [
      {
        "tag": "api",
        "id": "list_orders",
        "method": "GET",
        "path": "/api/v1/orders",
        "modifierTags": ["output"],
        "modifiers": { "bodyRef": "OrderListResponse" },
        "provenance": "Pactia"
      },
      { "tag": "auth", "roles": ["Operator"], "provenance": "Pactia" },
      { "tag": "api", "id": "create_order", "method": "POST", "path": "/api/v1/orders", "provenance": "Pactia" }
    ]
  }
}
```

### Slot — merge rules

| `ir.merge`         | Meaning                                                               |
| ------------------ | --------------------------------------------------------------------- |
| `append_host`      | One object appended to `body[]` (or parent host `body[]` when nested) |
| `merge_into_host`  | Modifier merges into the **next** host (e.g. `@@output` before `@api`) |
| `field_annotation` | Field-level tag merges under the owning model field line              |

Each appended host object carries **`tag`**, body fields, **`id`** or **`name`** when declared, optional **`modifierTags`** / **`modifiers`** on endpoints, and **`provenance`**.

Field lines in `@entity` bodies include **`modifierTags`** on each field (`pk`, `nullable`, `pii`, …) even when the modifier body is empty.

Modifier shorthand (`@@output VehicleListResponse` before `@api`) uses **`merge_into_host`** — see [language-spec.md — Tags](language-spec.md#tags).

Host-tag prefix shorthand (`@auth Customer` when the package def includes `modifier,`) lowers like the equivalent block body — see [language-spec.md — Tags](language-spec.md#tags).

### Prose

Standalone `>` / `>>` lines in a block lower as **`body[]`** entries (`kind: prose`). Prose inside a host tag block lowers into that host object's fields (`text`, `summary`) without a separate `body[]` slot.

### Context lowering

**Status: Implemented** — pactiac lowers `context { }` into **`context[]`** on the enclosing slice (structural — like `name` on product); pactia build writes the index and bundles files (see below).

The **`context`** keyword is **not** lowered through the tag registry. It does **not** use `tag` or `body[]`. A registry export `@context` (if any) still lowers to **`body[]`** with `tag: context` — no collision.

Each entry in **`context[]`**:

```json
{
  "name": "api_notes",
  "path": "./context/catalog/services/catalog-admin/api-notes.md",
  "guidance": ["API design notes for admin create flow."],
  "provenance": "Pactia"
}
```

`path` may be a string or array of strings as authored. Directory paths (trailing `/`) are stored as a single string; expansion to a file list happens in **pactia build**, not in pactiac.

**pactiac does not:** open files, validate paths exist, expand directories, compute digests, or copy assets.

---

## IR layout

IR file names mirror structural blocks: `<target>.<block>.json`.

| Pactia block                    | IR file               |
| ------------------------------- | --------------------- |
| `module trading { }`            | `trading.module.json` |
| `model { }` in module `trading` | `trading.model.json`  |
| `service OrderService { }`      | `order.service.json`  |

Example output tree (default `-o` layout):

```text
input/
  workspace.json          # single-file bundle — start here (agents, BSC)
  manifest.json           # compile index only
  product.json
  context.index.json      # optional — written by pactia build (digests, expanded directory files)
  context/                # optional — bundled copies by default; omitted with pactia build --no-bundle-context
  modules/
    trading/
      trading.module.json
      trading.model.json
      services/
        order.service.json
```

| Path                                    | Scope                                                                                                            |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `workspace.json`                        | **Full IR in one file** — `manifest`, `product`, and all module/model/service slices inline                      |
| `manifest.json`                         | Compile metadata and module file index (`pactiaVersion`, `entry`, `lockfileDigest`, `modules[]`, `references[]`) |
| `product.json`                          | Product-level lowered facts                                                                                      |
| `modules/<m>/<m>.module.json`           | Module scope                                                                                                     |
| `modules/<m>/<m>.model.json`            | Model scope                                                                                                      |
| `modules/<m>/services/<s>.service.json` | Service scope                                                                                                    |
| `body[]` on each slice above            | Ordered tags, prose, macro expansions in source order                                      |
| `context[]` on each slice above         | Context keyword attachments (`name`, `path`, optional `guidance`) — source order within `context[]` |

**Agents and tools:** read `workspace.json` first — it is the entry point referenced by kernel `@pactia`. Use per-slice files when you only need one module or service, or when diffing a single scope.

**Naming:** `<module>` kebab-case; `<service>` lowercased with optional `Service` suffix stripped (`OrderService` → `order.service.json`).

There is no JSON Schema for IR in the spec repo — only prose rules here and package `export def` field specs at compile time.

---

## `workspace.json` and `manifest.json`

**`workspace.json`** — single-file bundle of the entire compile output. Top-level keys:

| Key        | Contents                                                                                          |
| ---------- | ------------------------------------------------------------------------------------------------- |
| `manifest` | Same object as `manifest.json` — compile metadata and file index                                  |
| `product`  | Same object as `product.json` — product-level intent                                              |
| `modules`  | Array of `{ module, model, services[] }` slices — same data as the per-file JSON under `modules/` |

Emitted for agents and BSC: one read gives the full product without chasing paths.

**`manifest.json`** — the `manifest` slice alone. Records language version, entry file, lock digest, module file tree (`modules[].name`, `path`, `module`, `model`, `services[].file`), and cross-module `references[]`. Not product intent — navigation metadata only.

Products with **no package dependencies** may compile without a `pactia.lock`; `lockfileDigest` in manifest is then omitted.

---

## Workspace assembly

Compile from the project root (directory containing `product.pactia`).

**Folder-agnostic:** fragment locations come from `import … from ./path` in `product.pactia`. Directory names (`modules/`, `fragments/`, `domains/…`) are team convention only.

### Import + attach (normative)

1. Read `product.pactia` — product-level imports, fragment paths, attach tree.
2. Resolve each `import { symbols } from ./path.pactia` — load `export module`, `export service`, `export model`, `export context` bodies from those paths.
3. Collect package imports from all files (product + fragments) into the merged source.
4. Splice attach references: `module(name) { service(Symbol) { model(…) context(…) } }`.
5. Each file imports what it uses — see [language-spec.md — File-local imports](language-spec.md#file-local-imports).

Example:

```pactia
import { orders, orders_model, OrderService } from ./fragments/orders/…;

product Relay {
  #rust-stack
  module(orders) {
    service(OrderService) {
      model(orders_model)
    }
  }
}
```

Same mechanism when paths use `./modules/workspace/…` (see [PPM](https://github.com/pactia-lang/examples/tree/main/ppm)) — only import paths differ.

Diagnostics: `IMPORT_UNUSED`, `ATTACH_UNDEFINED`, `ATTACH_KIND_MISMATCH`, `CONTEXT_IMPORT_UNUSED`, `CONTEXT_ATTACH_UNDEFINED`, `CONTEXT_ATTACH_KIND_MISMATCH`.

Canonical attach example: [relay workspace](https://github.com/pactia-lang/pactiac/tree/main/test/fixtures/workspace/relay).

No module list in `pactia.toml`. Single-file products declare all blocks inline (see [relay.pactia monolith](https://github.com/pactia-lang/pactiac/blob/main/test/fixtures/kernel/relay.pactia)).

---

## Context index (pactia build)

**Status: Implemented** — `pactia build` walks compiled IR **`context[]`** on each slice and writes **`context.index.json`** beside the IR tree. Bundles files under `input/context/` by default; rewrites bundled `path` values to `context/...` locations so `out/` is self-contained for agents. Use `--no-bundle-context` to skip the copy and keep workspace-relative paths.

| Step | Behavior |
| ---- | -------- |
| Verify | Every file path exists; every directory path exists and is non-empty after expansion rules |
| Expand directory | Recursive walk; skip dotfiles and dot-directories; do not follow symlinks |
| Digest | `sha256:` per included file |
| Guardrails | Warn at **50** files per context block; error at **500** |
| Bundle (default) | Copy included files under `out/.../input/context/`; rewrite IR and index paths to bundle-relative locations; omit `package` on bundled index entries |

**Future:** `.pactiacontextignore` at workspace root (gitignore-style) to exclude paths from directory expansion — not required in v1.

Example index entry for a directory group:

```json
{
  "entries": [
    {
      "name": "admin_assets",
      "scope": "service/catalog/catalog-admin",
      "path": "context/catalog/services/catalog-admin/",
      "files": [
        { "path": "context/catalog/services/catalog-admin/api-notes.md", "digest": "sha256:…" }
      ]
    }
  ]
}
```

Agents and CI use digests in `context.index.json` to detect which attachments changed without diffing full IR slices.


## Provenance model

| Source          | Meaning                                                                                                              |
| --------------- | -------------------------------------------------------------------------------------------------------------------- |
| `Pactia`        | Written directly by the author                                                                                       |
| `INFERRED`      | Derived by a deterministic rule (**planned** — BSC or future pactiac pass; no normative inference rules in spec 1.2) |
| `PACKAGE`       | Supplied by an import                                                                                                |
| `MACRO`         | Supplied by `#macro` expansion                                                                                       |
| `DEFINE`        | Supplied by local `def #` template expansion                                                                         |
| `GUIDANCE`      | Prose (`>`, `>>`) or non-enforced hints                                                                              |
| `GENERATED`     | Optional `bsc expand` (LLM)                                                                                          |
| `NOT_DERIVABLE` | IR slot exists but source does not contain the fact                                                                  |

---

## Package publish

Packages ship **`pactia.toml` + `index.pactia`** plus all files referenced via `export "./file"`. Publish is a **git tag** on the package repo. See [packages.md](packages.md).

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
