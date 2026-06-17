# Pactia Compilation Guide

Status: **Specification** — Pactia 1.0 compiler pipeline.

Part of: [language-spec.md](language-spec.md) | [overview.md](overview.md#philosophy)

Pactia compiles to **AI-neutral YAML IR** (`input/**/*.yaml`) — not vendor-specific prompts. **BSC** renders that IR for Cursor, Claude Code, Copilot, or custom agents, and may **optionally use an LLM** to expand it into richer target-specific context — grounded in YAML, provenance `GENERATED`, never overriding formal IR facts. See [overview — AI model agnostic](overview.md#ai-native--and-ai-model-agnostic).

---

## Pipeline overview

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────────────────────┐
│  *.pactia    │────▶│  input/**/*.yaml    │────▶│  specification/            │
│  pactiac     │     │  (deterministic IR)  │     │  bsc render + expand (LLM) │
└──────────────┘     └──────────────────────┘     └──────────────────────────────┘
        │                        │
        │                        └── validated by @pactia/schema — source of truth
        └── stack package from @stack { } on product
```

`pactiac` never calls an LLM. Optional LLM expansion happens only in **BSC** after IR is fixed, and only for agent-facing narrative — not for mutating enforceable fields.

---

## Compile phases

```
0.  Assemble workspace: merge product.pactia, module.pactia, service.pactia,
    imported features/*.pactia and entities/*.pactia into single AST
1.  Read and validate version declaration (pactia 1.0)
2.  Lex: strip `//` line comments and `/* */` block comments (never in IR)
3.  Resolve @stack { }; fetch stack package; load stack macros[] into effectiveRegistry
4.  Resolve `use` / `import` per file scope; read `kabol.lock` pins for every package in `use`; build workspace effectiveRegistry
5.  Merge declarations (package AST fragments per exports flags → local imports → main file)
6.  Expand define (templates only) → kernel constructs
7.  Expand #[macro] using effectiveRegistry (stack > use > std > defaults)
8.  Validate @tag { } bodies against kernel rules + package JSON schemas
9.  Validate protocol package @grpc / @rest blocks against JSON schemas
10. Verify protocol packages against stack allowedProtocolPackages
11. Validate state graphs in data { states ... }
12. Infer missing response/request shapes; warn on ambiguity
13. Write kabol.lock if absent or updated
14. Lower @tags → YAML IR with provenance (MACRO, DEFINE, PACKAGE on expanded fields)
15. Lower @web { } / @ios { } → input/surfaces/*.yaml; resolve @bind { }
16. Apply yaml merge embeds: parse → validate → deep-merge (provenance: YAML_EMBED)
17. Write output files (optional: input/macros.yaml audit trail)
18. (optional) bsc compile-workspace → project-definition.yaml + specification/ (template render)
19. (optional) bsc expand --target cursor|claude-code|… → LLM-enriched agent context from IR (provenance GENERATED; cacheable)
```

See [language-spec.md](language-spec.md) for `yaml merge` rules and [workspace layout](language-spec.md#workspace-layout) for merge order.

---

## BSC render and expand (not pactiac)

| Step | Tool | LLM? | Output |
| --- | --- | --- | --- |
| Lower intent | `pactiac` | No | `input/**/*.yaml` |
| Template render | `bsc render` | No | `specification/*.md`, target rule files |
| Enrich for agent | `bsc expand` | Optional | Longer narratives, derived scenarios, stack hints — **grounded in IR** |

**Rules for `bsc expand`:**

- Input is **YAML IR only** — not raw `.pactia` re-interpretation.
- Must not add or alter facts with provenance `Pactia`, `MACRO`, `PACKAGE`, or `STACK_DEFAULT`.
- Elaborates `GUIDANCE`, `NOT_DERIVABLE`, and stack defaults into readable agent context.
- Target profile selects format (Cursor rules vs `CLAUDE.md` vs internal Copilot spec).
- Runs should be **cacheable** (same IR + target + model → same expand output) when reproducibility matters.

This is how a minimal Pactia file becomes a **contentful** agent brief without authors hand-writing 200 pages.

---

## Output file mapping

| Pactia source | Output file |
| --- | --- |
| `product { @stack @topology @tenancy ... }` | `input/project.yaml` |
| `service { @rest { } + nested @tag { } + #[macro] }` | `input/services/<kebab-name>.yaml` |
| `@actor { }` | `input/business.yaml` |
| `data { @entity @enum @relation @states }` | `input/domain.yaml` |
| `>` rules, constraint prose | `input/business.yaml` (prose / guidance) |
| `@integration` / integration prose | `input/integrations.yaml` |
| `@event { }` with `handler` line | `input/communication.yaml` + `eventHandlers[]` |
| `@policy { ... }` | `input/policies.yaml` |
| `@errorCatalog` | `input/errors.yaml` (catalog) |
| `@errors` on `@rest` | `response.errors[]` per endpoint |
| `@event` declarations | `input/communication.yaml` → `events[].payload` |
| `@config { ... }` | `input/config.yaml` |
| `@test` blocks | `input/scenarios.yaml` |
| `@must` blocks | `input/obligations.yaml` |
| Workspace `entities/*.pactia` | `input/domain.yaml` |
| Workspace `features/*.pactia` | `input/services/<kebab-name>.yaml` |
| `define template` (expands in place) | Same targets as expanded kernel — provenance `DEFINE` |
| `@web { }` / `@ios { }` surface blocks | `input/surfaces/*.yaml` |
| `@guide`, `>` (guidance) | `input/guidance.yaml` |
| `@security` | `input/security-policy.yaml` |
| `@observe` | `input/observability.yaml` |
| `@deploy` | `input/deployment.yaml` |
| `@compliance` | `policies.yaml` + domain annotations |

Service file naming: `FleetService` → `fleet-service.yaml`.

See [registry.md](registry.md#cross-cutting-concerns) for cascade and provenance.

---

## Stack resolution

```pactia
product FleetManagement {
  @stack { rust-anb ^1.0 }
}
```

1. Read `@stack { }` and optional semver constraint from `product`; expand bare id to `@pactia/<id>`.
2. Check `kabol.lock` — use cached copy when digest matches.
3. Otherwise query pactia.io for the highest matching release; write pin to `kabol.lock`.
4. Download, verify digest, cache locally.

Output in `input/project.yaml`:

```yaml
project:
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
| `PACKAGE` | Supplied by a `use` import (merged AST or registry) |
| `MACRO` | Supplied by `#[macro]` expansion |
| `DEFINE` | Supplied by `define template` expansion |
| `YAML_EMBED` | Supplied by a `yaml merge` embed |
| `GUIDANCE` | `@guide` or non-enforced prose |
| `NOT_DERIVABLE` | IR slot exists but Pactia does not contain the fact |

`NOT_DERIVABLE` entries are never enforced by conformance — they mark [the intent line](overview.md#the-intent-line).

---

## Package build pipeline (`pactia package build`)

Used by **package authors** before `pactia publish` — distinct from product compile.

```
1.  Read package source (index.pactia and/or pactia.package.yaml)
2.  Lower define macro { expands { } } → macros[] entries
3.  Lower define tag { scope body lowers } → tags[] + schemas/<name>-v1.json
4.  Compile kernel declarations (data, service, …) → domain/integrations fragments
5.  Merge yaml package/<section> heredocs → pactia.package.yaml
6.  Merge hand-authored macros[] / tags[] with lowered define macro / define tag
7.  Validate against schema for package.kind + @pactia/schema IR path allowlist
8.  Write publishable bundle + digest
```

Published tarball is what consumers load into `effectiveRegistry` at product compile time. See [packages.md](packages.md#package-registry-define-macro--define-tag).

---

## CLI

```bash
pactiac compile ../examples/single-file/fleet-management-v2.pactia -o ./input
pactiac compile -w ./my-product -o ./input
pactia check fleet-management-v2.pactia
pactia package build -C ./packages/rust-anb
pactia publish -C ./packages/rust-anb
pactia build fleet-management-v2.pactia -o ./specification
```

---

## See also

- [language-spec.md](language-spec.md)
- [platform.md](platform.md)
- [overview.md](overview.md#architecture-coverage)
- [language-spec.md](language-spec.md#workspace-layout)
