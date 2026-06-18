# Pactia Platform — Stacks, Versions, and Protocols

Version: **1.0**  
Status: **Specification**

Part of: [packages.md](packages.md) | [language-spec.md](language-spec.md) | [compilation.md](compilation.md)

Stack packages (`@stack`), semver locking, and wire-format protocol packages (`@pactia/protocol-rest`, `@pactia/protocol-grpc`, …). **`@api` is a kernel clause tag** — protocol packages add wire validation and nested blocks such as `@grpc { }`.

---

## Stack packages

## Selecting a stack

```pactia
product FleetManagement {
  > Fleet tracking platform

  @stack rust-anb { }
  @topology { mode: microservices, }
}
```

`@stack <id> { }` selects a stack package. The compiler resolves a bare id as `@pactia/<id>`. For a private or scoped stack use the full coordinate as the tag target:

```pactia
product OrderPlatform {
  @stack @acme/node-bff { }
}
```

| Syntax | Resolves to |
| --- | --- |
| `@stack rust-anb { }` | `@pactia/rust-anb` — version from `pactia.toml` / `pactia.lock` |
| `@stack @acme/node-bff { }` | Private package at configured registry |

**Do not** put semver inside `@stack { }`. Constraints live in `pactia.toml`; pins in `pactia.lock`. `VERSION_IN_TAG_BODY` if `version:` appears in the tag body. See [packages.md — @stack vs import](packages.md#stack-vs-import-for-stack-kind-packages-eg-rust-anb).

---

## Stack package manifest

Stack packages use the standard `pactia.package.yaml` with `kind: stack`:

```yaml
package:
  name: "@pactia/rust-anb"
  version: 1.0.0
  description: Rust / Axum / PostgreSQL / Redis / Kafka / Kubernetes
  license: MIT
  kind: stack
  pactiaVersion: "1.0"

profile:
  stack:
    language: rust
    framework: axum
    database: postgresql
    cache: redis
    events: kafka
    container: kubernetes

  architecture:
    pattern: clean-architecture
    layers: [domain, application, infrastructure, api]
    sync: http-rest
    async: kafka

  policy:
    auth: jwt
    tokenTtl: 3600
    defaultRetention: 7y
    residency: unrestricted

  deployment:
    port: 8080
    healthPath: /health/live

  codingStandards:
    - "Use sqlx query! macro for all database queries"
    - "Map all errors to the platform error envelope before returning"
    - "No unwrap() in handler code; use ? with typed errors"
    - "rustls only — no openssl or native-tls"

platformLaw:
  errorEnvelope: { code: string, message: string, requestId: string }
  pagination: { style: CURSOR, defaultLimit: 20, maxLimit: 100 }
  kafkaNaming: { pattern: "<domain>.<entity>.<event>", prefix: "" }
  cacheKeyPattern: "<service>:<entity>:<id>"
  jwtClaims: [sub, role, iat, exp]

technologyPolicy:
  use:
    language:   [{ id: rust, version: ">=1.78", reason: "Edition 2021; stable async" }]
    framework:  [{ id: axum, version: "^0.7" }]
    database:   [{ id: postgresql, driver: sqlx, migrations: sqlx-cli }]
    cache:      [{ id: redis, client: redis-rs }]
    events:     [{ id: kafka, client: rdkafka, serialization: json }]
    auth:       [{ id: jwt, crate: jsonwebtoken }]
    tls:        [{ id: rustls }]
    observability: [{ id: opentelemetry }, { id: prometheus }]
  forbid:
    frameworks: [{ id: actix-web, reason: "Platform standard is Axum" }]
    databases:  [{ id: mongodb, reason: "Relational model required" }]
    patterns:   [{ id: unwrap-in-handlers, reason: "Map errors to error envelope" }]
    crates:     [{ id: openssl, reason: "rustls only" }]
  workspaceCrates:
    axum: "0.7"
    sqlx: "0.8"
    tokio: "1"
    rdkafka: "0.36"

allowedProtocolPackages:
  - coordinate: "@pactia/protocol-rest"
    version: "^1.0"
```

Every `forbid` entry must include `reason` so architects and AI understand _why_, not just _what_.

---

## What products can override

Stack packages define platform law. Products select and narrow — they do not replace individual decisions.

| Override in Pactia | Allowed? |
| --- | --- |
| `@policy { retain Entity 7y }` | Yes — narrows retention for one entity |
| `@deploy { @environment production { replicas: 5 } }` | Yes — scale per environment |
| `@stack rust-anb { }` → `@stack @acme/node-bff { }` | Yes — pick a different stack package |
| Importing a crate forbidden by the stack | No — `technologyPolicy.forbid` is enforced by conformance |
| Overriding `platformLaw.errorEnvelope` shape | No — platform law is locked |

---

## Publishing a stack package

Stack packages are **YAML-native**. The kernel has no `profile` or `platformLaw` syntax — authors use `yaml package/<section>` blocks in `index.pactia`, or hand-author `pactia.package.yaml` directly. See [packages.md](packages.md) (authoring) and [language-spec.md — `yaml` embed](language-spec.md#yaml-embed).

### Author with `yaml` blocks (recommended)

```pactia
pactia 1.0
// Package: @pactia/rust-anb

yaml package """
package:
  name: "@pactia/rust-anb"
  version: 1.0.0
  kind: stack
  pactiaVersion: "1.0"
  license: MIT
  description: Rust / Axum / PostgreSQL / Redis / Kafka / Kubernetes
"""

yaml package/profile """
stack:
  language: rust
  framework: axum
  database: postgresql
  cache: redis
  events: kafka
  container: kubernetes
architecture:
  pattern: clean-architecture
  layers: [domain, application, infrastructure, api]
  sync: http-rest
  async: kafka
policy:
  auth: jwt
  tokenTtl: 3600
  defaultRetention: 7y
  residency: unrestricted
deployment:
  port: 8080
  healthPath: /health/live
codingStandards:
  - "Use sqlx query! macro for all database queries"
  - "Map all errors to the platform error envelope before returning"
  - "No unwrap() in handler code; use ? with typed errors"
  - "rustls only — no openssl or native-tls"
"""

yaml package/platformLaw """
errorEnvelope:
  code: string
  message: string
  requestId: string
pagination:
  style: CURSOR
  defaultLimit: 20
  maxLimit: 100
kafkaNaming:
  pattern: "<domain>.<entity>.<event>"
  prefix: ""
cacheKeyPattern: "<service>:<entity>:<id>"
jwtClaims: [sub, role, iat, exp]
"""

yaml package/technologyPolicy """
use:
  language: [{ id: rust, version: ">=1.78", reason: "Edition 2021; stable async" }]
  framework: [{ id: axum, version: "^0.7" }]
  database: [{ id: postgresql, driver: sqlx, migrations: sqlx-cli }]
  cache: [{ id: redis, client: redis-rs }]
  events: [{ id: kafka, client: rdkafka, serialization: json }]
  auth: [{ id: jwt, crate: jsonwebtoken }]
  tls: [{ id: rustls }]
  observability: [{ id: opentelemetry }, { id: prometheus }]
forbid:
  frameworks: [{ id: actix-web, reason: "Platform standard is Axum" }]
  databases: [{ id: mongodb, reason: "Relational model required" }]
  patterns: [{ id: unwrap-in-handlers, reason: "Map errors to error envelope" }]
  crates: [{ id: openssl, reason: "rustls only" }]
workspaceCrates:
  axum: "0.7"
  sqlx: "0.8"
  tokio: "1"
  rdkafka: "0.36"
"""

yaml package/allowedProtocolPackages """
- coordinate: "@pactia/protocol-rest"
  version: "^1.0"
"""
```

### Build and publish

```bash
pactia package init @pactia/rust-anb --kind stack   # or clone examples/packages/rust-anb
pactia package build                                 # merge yaml fragments → pactia.package.yaml
pactia package check                                 # validate against stack package schema
pactia publish                                       # upload to pactia.io
```

### Consumer selection

```pactia
product FleetManagement {
  > Fleet tracking platform

  @stack rust-anb { }
  @topology { mode: microservices, }
}
```

The compiler resolves the package named by the `@stack` tag target through the same [package resolver](packages.md#package-resolution) as `import`. When the tarball has `kind: stack`, it merges `profile` + `platformLaw` + `technologyPolicy` into workspace IR and registers stack-tier macros (e.g. `#[database]`). Std packages such as `@pactia/protocol-rest` still require explicit `import` — add `import @pactia/protocol-rest;` before using REST wire fields on `@api`. No local `stacks/` directory is needed.

## `@stack` tag lowering

`@stack` is validated and lowered like any other kernel clause tag on `product`. Package fetch, semver, and lock checks use generic `PACKAGE_*` codes — not stack-specific errors.

1. Parse `@stack <target> { }` from `product`; expand bare target id to `@pactia/<id>`.
2. Resolve coordinate via `pactia.toml` / `pactia.lock` (same path as `import @scope/name`).
3. Verify `pactia.toml [stack].package` matches the resolved coordinate (`STACK_BINDING_MISMATCH` if not).
4. When `kind: stack`, merge platform law into workspace IR and register package `tags[]` / `macros[]`.
5. Emit `input/product.yaml` with `stackId`, `stackVersion`, `stackDigest`.
6. Emit module-scoped IR under `input/modules/<module>/`.
7. Conformance (planned): flag generated code that imports forbidden crates.

---

## rust-anb quick reference

| Use | Do not use |
| --- | --- |
| Rust ≥ 1.78, Axum 0.7 | Actix, Rocket, Warp |
| PostgreSQL + sqlx (`query!`) | MongoDB, MySQL, raw string SQL |
| Redis (redis-rs) | In-process cache as system of record |
| Kafka (rdkafka), JSON events | RabbitMQ, in-memory event bus in prod |
| JWT (jsonwebtoken + tower) | Session cookies without RFC |
| OpenTelemetry + Prometheus | Ad-hoc println logging |
| Kubernetes + Helm baseline | Docker-compose as production target |
| rustls TLS | openssl / native-tls |

---

## See also

- [examples/packages/rust-anb/](examples/packages/rust-anb/) — reference stack package source
- [Stack versions](#stack-versions) — semver, pactia.lock, migration
- [packages.md](packages.md) — authoring and publishing all package kinds
- [overview.md](overview.md#architecture-coverage) — stack vs Pactia vs compiler
- [packages.md](packages.md#extensibility) — packages extend domain, not stack
- [language-spec.md](language-spec.md) — `@stack` on `product`

---

## Stack versions

## Stack vs product version

| Field | Where | Meaning |
| --- | --- | --- |
| Product release semver | Git tags / release process (not a kernel field in 1.0) | Your product version |
| `package.version` in `pactia.package.yaml` | Stack package manifest | Platform stack semver |
| `[stack]` / `[dependencies]` in `pactia.toml` | Project manifest | Semver **constraint** on which stack release to load |
| `pactia.lock` | Lock file | **Pinned** stack version + digest |

Do not confuse product release versioning with stack package semver. They are independent.

---

## Selecting a stack

```
StackDecl ::= "@stack" StackCoordinate "{" StackBody "}"
StackCoordinate ::= Identifier | "@" Identifier "/" Identifier
StackBody       ::= ( AssignmentLine | ProseLine )*
```

```pactia
product FleetManagement {
  @stack rust-anb { }              // resolves to @pactia/rust-anb
  @stack @acme/node-bff { }        // private or org-scoped stack
}
```

| Form | Resolves to |
| --- | --- |
| `@stack rust-anb { }` | `@pactia/rust-anb` — version from `pactia.toml` / `pactia.lock` |
| `@stack @acme/node-bff { }` | Package `@acme/node-bff` from configured registry |

Bare ids resolve to `@pactia/<id>` on pactia.io. Semver constraints and exact pins belong in `pactia.toml`, not in the `@stack` tag body.

```toml
# pactia.toml
[stack]
package = "@pactia/rust-anb"

[dependencies]
"@pactia/rust-anb" = "^1.0"    # >=1.0.0 <2.0.0
# "@pactia/rust-anb" = "1.0.0" # exact pin
```

### Compilation output

```yaml
# input/product.yaml
product:
  stackId: "@pactia/rust-anb"
  stackVersion: 1.0.0
  stackDigest: sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

---

## Lock file: `pactia.lock`

Stack packages are pinned in the same `pactia.lock` as domain and protocol packages — there is no separate `pactia.lock`.

```yaml
lockVersion: 1
packages:
  - name: "@pactia/rust-anb"
    kind: stack
    version: 1.0.0
    digest: sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
    resolvedAt: 2026-06-10T12:00:00Z
  - name: "@pactia/kyc-compliance"
    kind: domain
    version: 1.0.3
    digest: sha256:...
    resolvedAt: 2026-06-10T12:00:00Z
```

| Command | Behavior |
| --- | --- |
| `pactiac compile` | Uses `pactia.lock` if present; else resolves and writes lock |
| `pactia update` | Re-resolve all packages within declared constraints; refresh lock |
| `pactia update --stack` | Re-resolve only stack package |

CI should commit `pactia.lock`. AI agent sessions must not silently upgrade platform law.

---

## Semver policy for stack packages

Stack package semver follows **platform impact**, not crate patch bumps alone.

| Bump | When | Examples |
| --- | --- | --- |
| **MAJOR** | Breaking change for existing products or generated specs | Rename `platformLaw.errorEnvelope` fields; switch auth type; remove a `technologyPolicy.use` requirement |
| **MINOR** | Additive, backward-compatible | New optional `technologyPolicy.forbid` entry; new `profile.deployment` field; additional crate pin |
| **PATCH** | No semantic change to platform law | README only; typo in `reason` strings; patch crate version within same API |

### What triggers a MAJOR bump

- `platformLaw` shape change consumed by reconciler or OpenAPI generator
- Default pagination style change (`CURSOR` → `OFFSET`)
- Kafka serialization change (`JSON` → `AVRO`)
- Required JWT claims set change

### What does NOT require a MAJOR bump

- Adding forbidden patterns in `technologyPolicy.forbid`
- Tightening `workspaceCrates` within same major crate line
- New stack package id (`@pactia/rust-minimal`) — that is a new package, not a new version of `@pactia/rust-anb`

---

## Compatibility

Stack packages declare the Pactia versions they support in their manifest:

```yaml
package:
  name: "@pactia/rust-anb"
  version: 1.0.0
  kind: stack
  pactiaVersion: "1.0"
```

Domain and protocol packages declare compatible stacks:

```yaml
package:
  name: "@pactia/kyc-compliance"
  kind: domain
  compatibleStacks:
    - name: "@pactia/rust-anb"
      version: "^1.0"
```

Compiler error `COMPATIBLE_STACKS_UNSATISFIED` if the product's resolved stack-kind package does not satisfy an imported package's `compatibleStacks`.

---

## Available stacks (pactia.io)

| Package | Latest | Pactia | Summary |
| --- | --- | --- | --- |
| `@pactia/rust-anb` | 1.0.0 | 1.0 | Rust, Axum, PostgreSQL, Redis, Kafka, K8s |

Planned:

| Package | Purpose |
| --- | --- |
| `@pactia/rust-minimal` | Rust + Axum + PostgreSQL only (no Kafka) |
| `@pactia/node-bff` | TypeScript BFF in front of rust-anb services |

---

## Migrating between stack versions

1. Read the package changelog on pactia.io.
2. Update `pactia.toml` constraint (e.g. `"@pactia/rust-anb" = "^1.1"`).
3. Run `pactia update --stack` or delete `pactia.lock` and recompile.
4. Review diff in IR slices and `platformLaw` sections.
5. Run conformance tests against forbidden crate list if enabled.

**Cross-major migration** (`^1.0` → `^2.0`) may require product `policy` or endpoint changes — treat as a platform project, not a silent dependency bump.

---

## Compile errors

`@stack` uses the same [package resolution](packages.md#package-resolution) errors as `import`. The only stack-specific code is **`STACK_BINDING_MISMATCH`** — when the `@stack` tag target, optional matching `import`, and `pactia.toml [stack].package` disagree.

Full implementer list: [grammar-reference.md — Package resolution](grammar-reference.md#package-resolution).

---

## CLI

```bash
pactia stack list                    # packages with kind:stack on pactia.io
pactia stack show @pactia/rust-anb   # versions, technologyPolicy summary
pactia stack resolve rust-anb ^1.0   # print resolved version + digest
pactia update --stack                 # refresh stack pin in pactia.lock
```

---

## See also

- #stack-packages — stack package anatomy, use/forbid policy
- [packages.md](packages.md) — all package types and pactia.lock
- [language-spec.md](language-spec.md) — `@stack` on `product`
- [compilation.md](compilation.md) — package resolution during compile

---

## Protocol packages

## One sentence

**Declare each API with the kernel clause `@api { }`; prefix `@auth { }`, `#[macro]`, `@returns`, and `@throws` on lines above it. Nest protocol-specific blocks (`@grpc { }`, `@graphql { }`) inside `@api { }` when needed. Import `@pactia/protocol-rest` for REST wire validation (`method`, `path`). The compiler lowers to YAML.**

---

## Why not protocol keywords

| Anti-pattern | Problem |
| --- | --- |
| `rest`, `grpc`, `graphql` as Pactia keywords | Language must change for every new protocol |
| `GET`, `POST` as kernel syntax | Couples compiler to HTTP |
| Fixed protocol enum on `product` | Does not scale |
| Separate grammar per protocol | Duplicated auth, DTOs, AI confusion |

| Correct pattern | Benefit |
| --- | --- |
| `import @pactia/protocol-rest;` | REST is a library |
| `@api list_vehicles { method: GET, path: "/api/v1/vehicles", … }` | Wire format is a tag block |
| `import @vendor/protocol-odata;` | New protocol without a language version bump |

---

## Layers

```
┌─────────────────────────────────────────────────────────┐
│  Pactia KERNEL (nine keywords + @tag { } + #[macro] + prose) │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│  PROTOCOL PACKAGES (pactia.io)                          │
│  @pactia/protocol-rest   → wire schema for @api fields  │
│  @pactia/protocol-grpc   → @grpc { } + schema + YAML   │
└───────────────────────────┬─────────────────────────────┘
                            │ compile
┌───────────────────────────▼─────────────────────────────┐
│  BSC YAML IR (input/modules/*/*.module.yaml, *.model.yaml, *.service.yaml) │
└─────────────────────────────────────────────────────────┘
```

---

## REST (default)

```pactia
import @pactia/protocol-rest;

service FleetService {
  @auth { roles: [Customer, Admin] }
  #[list]
  #[paginated]
  #[owner]
  @returns VehicleListResponse
  @throws { names: [Forbidden] }
  @api list_vehicles {
    method: GET,
    path: "/api/v1/vehicles",
  }
}
```

`@api { }` is **required** for REST endpoints. Import `@pactia/protocol-rest` for wire-format validation and OpenAPI-oriented defaults.

---

## gRPC and GraphQL

Attach a protocol block to the same logical `@api { }` operation:

```pactia
import @pactia/protocol-grpc;

@auth { roles: [Trader] }
#[buyer]
@body MarkPaymentSentRequest
@returns TradeResponse
@api mark_payment_sent {
  method: POST,
  path: "/api/v1/trades/:id/mark-payment-sent",

  @grpc {
    service: trade.TradeService,
    rpc: MarkPaymentSent,
  }
}
```

`@api` is always a **kernel clause tag**. Nested block names such as `@grpc { }` and `@graphql { }` come from the protocol package manifest — not from the kernel.

---

## Publishing a new protocol

1. `pactia package init @yourorg/protocol-foo --kind protocol`
2. Author `index.pactia` with `yaml package` and `yaml package/extensions` blocks
3. Add `schemas/<name>-v1.json` for each extension schema
4. `pactia package build` → validate manifest + schemas
5. `pactia publish` → pactia.io
6. Consumers: `pactia add @yourorg/protocol-foo@^1.0` then `import @yourorg/protocol-foo;`

See [packages.md](packages.md). New wire formats are **packages**, not kernel keywords.

---

## See also

- [language-spec.md](language-spec.md) — strict kernel, `@api { }`, model tags
- [packages.md](packages.md#extensibility)
- [packages.md](packages.md)
- [compilation.md](compilation.md)
