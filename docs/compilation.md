# Pactia Compilation Guide

Status: **Specification** — Pactia 1.0 compiler pipeline.

Part of: [language-spec.md](language-spec.md) | [overview.md](overview.md#philosophy)

Pactia compiles to **AI-neutral YAML IR** (`input/**/*.yaml`) — not vendor-specific prompts. **BSC** renders that IR for Cursor, Claude Code, Copilot, or custom agents, and may **optionally use an LLM** to expand it into richer target-specific context — grounded in YAML, provenance `GENERATED`, never overriding formal IR facts. See [overview — AI model agnostic](overview.md#ai-native--and-ai-model-agnostic).

---

## Pipeline overview

```
┌──────────────┐     ┌──────────────────────────────┐     ┌──────────────────────────────┐
│  *.pactia    │────▶│  input/                      │────▶│  bsc render + expand (LLM)   │
│  pactiac     │     │  *.module.yaml *.model.yaml  │     │  (optional agent briefs)     │
│              │     │  *.service.yaml              │     │                              │
└──────────────┘     └──────────────────────────────┘     └──────────────────────────────┘
```

`pactiac` never calls an LLM. Optional LLM expansion happens only in **BSC** after IR is fixed, and only for agent-facing narrative — not for mutating enforceable fields.

---

## Compile phases

```
0.  Assemble workspace: merge product.pactia, module.pactia, service.pactia,
    imported features/*.pactia and entities/*.pactia into single AST
1.  Read and validate version declaration (pactia 1.0)
2.  Lex: strip `//` line comments and `/* */` block comments (never in IR)
3.  Resolve all package coordinates (`import` lines and `@stack` tag targets) via `pactia.toml` / `pactia.lock`; build effectiveRegistry
4.  Merge declarations (package AST fragments per exports flags → local imports → main file)
5.  Expand define (templates only) → kernel constructs
6.  Expand #[macro] using effectiveRegistry (stack-tier package > explicit import > std > defaults)
7.  Validate @tag { } bodies against kernel rules + package JSON schemas
8.  Validate @api wire fields (when `import @pactia/protocol-rest`) and protocol-package nested blocks (@grpc, @graphql, …) against JSON schemas
9.  Verify protocol packages against stack allowedProtocolPackages
10. Validate state graphs in model { states ... }
11. Infer missing response/request shapes; warn on ambiguity
12. Write pactia.lock if absent or updated
13. Lower @tags → YAML IR with provenance (MACRO, DEFINE, PACKAGE on expanded fields)
14. Lower `@surface { }` → `product.yaml` (`surfaces[]`); resolve `@bind { }`
15. Apply yaml merge embeds: parse → validate → deep-merge (provenance: YAML_EMBED)
16. Write module-scoped output: `<module>.module.yaml`, `<module>.model.yaml`, `services/<service>.service.yaml`
17. (optional) `bsc render` → agent briefs from module-scoped IR (`input/modules/*/*.yaml`)
18. (optional) bsc expand --target cursor|claude-code|… → LLM-enriched agent context (provenance GENERATED; cacheable)
```

See [language-spec.md](language-spec.md) for `yaml merge` rules and [workspace layout](language-spec.md#workspace-layout) for **workspace file merge order**.

### State graph validation (phase 10)

After tag JSON Schema validation, the compiler validates every `@states { }` block in each `model { }` and every `@transition { }` on `@api` endpoints in the same module.

**Binding.** `@states` declares `entity: EntityName.fieldName`. The compiler resolves `EntityName` to an `@entity` in the same module and `fieldName` to a field on that entity. The field's type must name an `@enum` declared in the same module.

**Transition values.** Each `from` and `to` in `@states transitions: [...]` must be a member of that enum's `values` list.

**Graph shape.**

| Rule | Code |
| --- | --- |
| Duplicate `(from, to)` edge in the same `@states` block | `STATE_DUPLICATE_TRANSITION` |
| Two `@states` blocks bind the same `entity.field` | `STATE_MACHINE_DUPLICATE` |
| Unknown entity, field, enum, or enum member | `STATE_BINDING_INVALID` |
| `@transition { from, to }` on `@api` references an edge not declared in any `@states` in the module | `STATE_TRANSITION_UNDEFINED` |
| `@transition` uses enum members invalid for the bound enum | `STATE_BINDING_INVALID` |

**Not validated in v1:** reachability analysis, initial-state requirements, or matching `@transition` to a specific `@states` id when multiple machines exist — only that the edge exists in at least one module `@states` graph and all values are enum members.

**Two merge pipelines:** [Compile merge order](language-spec.md#compile-merge-order) assembles multi-file workspaces (`product.pactia`, `module.pactia`, `features/*.pactia`, …). [Package merge order](#package-merge-order) below overlays imported domain-package AST. The [language-spec compile pipeline](language-spec.md#compile-pipeline) is an abbreviated author-facing summary; this section is the implementer reference.

---

## BSC render and expand (not pactiac)

| Step | Tool | LLM? | Output |
| --- | --- | --- | --- |
| Lower intent | `pactiac` | No | `input/**/*.yaml` |
| Template render | `bsc render` | No | Agent briefs, target rule files |
| Enrich for agent | `bsc expand` | Optional | Longer narratives, derived scenarios, stack hints — **grounded in IR** |

**Rules for `bsc expand`:**

- Input is **YAML IR only** — not raw `.pactia` re-interpretation.
- Must not add or alter facts with provenance `Pactia`, `MACRO`, `PACKAGE`, or `STACK_DEFAULT`.
- Elaborates `GUIDANCE`, `NOT_DERIVABLE`, and stack defaults into readable agent context.
- Target profile selects format (Cursor rules vs `CLAUDE.md` vs internal Copilot spec).
- Runs should be **cacheable** (same IR + target + model → same expand output) when reproducibility matters.

This is how a minimal Pactia file becomes a **contentful** agent brief without authors hand-writing 200 pages.

---

## IR layout

IR file names **mirror the kernel block** they lower from: `<target>.<block>.yaml`.

| Pactia block | IR file |
| --- | --- |
| `module trading { }` | `trading.module.yaml` |
| `model { }` in module `trading` | `trading.model.yaml` |
| `service OrderService { }` | `order.service.yaml` |

Ownership chain:

```
Manifest
 └─ Product
      └─ Module
           ├─ Model
           └─ Service
```

Example output tree (`pactiac compile -w ./my-product -o ./input`):

```text
input/
  manifest.yaml
  product.yaml

  modules/
    trading/
      trading.module.yaml
      trading.model.yaml
      services/
        order.service.yaml

    wallet/
      wallet.module.yaml
      wallet.model.yaml
      services/
```

| Path | Scope | Contents |
| --- | --- | --- |
| `manifest.yaml` | Compile output | Version, entry, lockfile digest, module file index, cross-module `references` (see below) |
| `product.yaml` | Product | `product { }` — `@stack`, `@topology`, `@tenancy`, `@guide`, **`surfaces`**, **`security`**, **`deployment`** |
| `modules/<module>/<module>.module.yaml` | Module | `module { }` — `@actor`, `@event`, `@config`, `@errors`, `@integration`, rules, `depends_on` |
| `modules/<module>/<module>.model.yaml` | Module | `model { }` — `@entity`, `@enum`, `@relation`, `@states`, model rules |
| `modules/<module>/services/<service>.service.yaml` | Service | `service { }` — metadata, `@api { }`, `@test`, `@must`, nested API tags |

**Naming rules:**

- `<module>` — kebab-case of the `module` clause identifier (`trading` → folder `modules/trading/`, files `trading.module.yaml`, `trading.model.yaml`).
- `<service>` — service clause identifier lowercased, with a trailing `Service` suffix removed when present (`OrderService` → `order.service.yaml`, `FleetService` → `fleet.service.yaml`).

**Agent retrieval:** work on `OrderService` in module `trading` → read `modules/trading/trading.module.yaml`, `modules/trading/trading.model.yaml`, and `modules/trading/services/order.service.yaml`.

Single-file products with one `module` block compile to the same shape under `modules/<module>/`. Products with no explicit `module` use `modules/default/default.module.yaml` and `default.model.yaml`.

See [registry.md](registry.md#cross-cutting-concerns) for tag-level routing and provenance.

---

## `manifest.yaml`

`manifest.yaml` records **what the compiler produced**: language version, compile metadata, the module file tree, and cross-module model links. It is emitted last.

It is **not** product intent — no tags, no macros, no `@security` / `@deploy` / surface facts (those lower to `product.yaml`).

| Field | Source |
| --- | --- |
| `pactiaVersion` | `pactia 1.0` declaration |
| `compiledAt` | Compiler timestamp (ISO 8601) |
| `entry` | Workspace entry file (e.g. `product.pactia`) |
| `lockfileDigest` | Digest of `pactia.lock` at compile time |
| `modules[]` | One entry per compiled module — file paths only |
| `references[]` | Cross-module `@fk` / `@relation` edges the compiler resolved |

Module and service file names in `modules[]` are **relative to** `path` (the module directory).

```yaml
manifest:
  pactiaVersion: "1.0"
  compiledAt: "2026-06-18T12:00:00Z"
  entry: product.pactia
  lockfileDigest: sha256:abc123...

  modules:
    - name: trading
      path: modules/trading/
      module: trading.module.yaml
      model: trading.model.yaml
      services:
        - name: order
          file: services/order.service.yaml

    - name: wallet
      path: modules/wallet/
      module: wallet.module.yaml
      model: wallet.model.yaml
      services: []

  references:
    - from:
        module: trading
        entity: Order
        field: customerId
      to:
        module: identity
        entity: Customer
```

`references[]` entries are derived from compiled `*.model.yaml` — e.g. a field with `@fk { entity: Customer }` where `Customer` is declared in another module. Keys `module`, `entity`, and `field` are manifest schema fields, not kernel keywords.

---

## Output file mapping

Paths are relative to the compile output root (`input/` by convention). `<module>` and `<service>` are kebab-case.

| Pactia source | Output file |
| --- | --- |
| Compiler-emitted file index | `manifest.yaml` |
| `product { @stack @topology @tenancy ... }` | `product.yaml` |
| `module { @actor @event @config @errors ... }` | `modules/<module>/<module>.module.yaml` |
| `model { @entity @enum @relation @states }` | `modules/<module>/<module>.model.yaml` |
| `>` rules on `module` | `<module>.module.yaml` (`rules[]` / guidance) |
| `>` rules on `model` | `<module>.model.yaml` |
| `@integration` | `<module>.module.yaml` (`integrations[]`) |
| `@event { }` with `handler` line | `<module>.module.yaml` (`events[]`, `eventHandlers[]`) |
| `@observe` on module | `<module>.module.yaml` |
| `service { @api { } + nested @tag { } + #[macro] }` | `modules/<module>/services/<service>.service.yaml` |
| `@throws` on `@api` | `endpoint.errors` in the service file |
| `@test` blocks | `<service>.service.yaml` (`scenarios[]`) |
| `@must` blocks | `<service>.service.yaml` (`obligations[]`) |
| Workspace `entities/*.pactia` | `<module>.model.yaml` (aggregated per module) |
| Workspace `features/*.pactia` | `<service>.service.yaml` |
| `define template` (expands in place) | Same targets as expanded kernel — provenance `DEFINE` |
| `@surface { }` blocks | `product.yaml` (`surfaces[]`) |
| `@guide` on product | `product.yaml` |
| `@guide` on module / service | Respective `.module.yaml` or `.service.yaml` |
| `@security`, `@policy`, `@compliance` | `product.yaml` (`security`) |
| `@deploy`, `@environment`, `@gate` | `product.yaml` (`deployment`) |
| `@compliance` field annotations | `<module>.model.yaml` field annotations |
| Cross-module `@fk` / `@relation` | `*.model.yaml` + `manifest.yaml` `references[]` |

---

## `@stack` tag lowering

```pactia
product FleetManagement {
  @stack rust-anb { }
}
```

`@stack` is a kernel clause tag on `product`. The compiler resolves its **target** coordinate through the same [package resolver](packages.md#package-resolution) as `import` — no dedicated stack phase.

1. Expand bare `@stack` target to `@pactia/<id>` when unqualified.
2. Resolve semver from `pactia.toml`; use `pactia.lock` pin when present.
3. Verify `pactia.toml [stack].package` matches (`STACK_BINDING_MISMATCH` if not).
4. Download tarball, verify digest, cache locally.
5. When `kind: stack`, merge platform law and register package macros.

Output in `input/product.yaml`:

```yaml
product:
  stackId: "@pactia/rust-anb"
  stackVersion: 1.0.0
  stackDigest: sha256:...
```

See [platform.md](platform.md#stack-versions).

---

## Package merge order

```
@pactia/audit-trail (transitive deps, deepest first)
@pactia/kyc-compliance
./local/overrides.pactia
main.pactia              ← wins on intentional override; warning emitted
```

Name collisions without alias → `EXPORT_COLLISION` error.

---

## Provenance model

| Source | Meaning |
| --- | --- |
| `Pactia` | Written directly by the author |
| `INFERRED` | Derived by a documented deterministic rule |
| `STACK_DEFAULT` | Supplied by the stack package |
| `PACKAGE` | Supplied by an `import` (merged AST or registry) |
| `MACRO` | Supplied by `#[macro]` expansion |
| `DEFINE` | Supplied by `define template` expansion |
| `YAML_EMBED` | Supplied by a `yaml merge` embed |
| `GUIDANCE` | `@guide` or non-enforced prose |
| `GENERATED` | Optional `bsc expand` (LLM) narrative from IR |
| `NOT_DERIVABLE` | IR slot exists but Pactia does not contain the fact |

`NOT_DERIVABLE` entries are never enforced by conformance — they mark [the intent line](overview.md#the-intent-line).

---

## Package build pipeline (`pactia package build`)

Used by **package authors** before `pactia publish` — distinct from product compile.

```
1.  Read package source (index.pactia and/or pactia.package.yaml)
2.  Lower define macro { expands { } } → macros[] entries
3.  Lower define tag { scope body lowers } → tags[] + schemas/<name>-v1.json
4.  Compile kernel declarations (model, service, …) → `*.model.yaml` / `*.module.yaml` fragments
5.  Merge yaml package/<section> heredocs → pactia.package.yaml
6.  Merge hand-authored macros[] / tags[] with lowered define macro / define tag
7.  Validate against schema for package.kind + @pactia/schema IR path allowlist
8.  Write publishable bundle + digest
```

Published tarball is what consumers load into `effectiveRegistry` at product compile time. See [packages.md](packages.md#package-registry-define-macro--define-tag).

---

## CLI

```bash
pactiac compile ../fixtures/kernel/fleet-management-v2.pactia -o ./input
pactiac compile -w ./my-product -o ./input
pactia check fleet-management-v2.pactia
pactia package build -C ./packages/rust-anb
pactia publish -C ./packages/rust-anb
pactia build fleet-management-v2.pactia -o ./specification
```

---

## Error codes

**Author-facing** (beginners at altitude 0–1): see [language-spec.md — Author errors](language-spec.md#author-errors).

**Implementer-facing** (registry collision, decorator placement, clause validation): see [grammar-reference.md — Implementer error codes](grammar-reference.md#implementer-error-codes).

---

## See also

- [language-spec.md](language-spec.md)
- [platform.md](platform.md)
- [overview.md](overview.md#architecture-coverage)
- [language-spec.md](language-spec.md#workspace-layout)
