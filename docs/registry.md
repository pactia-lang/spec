# Pactia Registry — Tags, Macros, and Cross-Cutting Blocks

Version: **1.0**  
Status: **Specification**

> **If you are not importing packages, skip this document** — kernel tags and macros are always available with no setup. Return when you add `import @scope/name;` or publish packages.

Part of: [language-spec.md](language-spec.md) | [packages.md](packages.md) | [compilation.md](compilation.md)

Single reference for **`@tag { }`**, **`#[macro]`**, and policy / security / observe / deploy blocks when you use packages or need the full tag matrix.

**Registry model:** tags and macros are **workspace-scoped** — see [Workspace registry](#workspace-registry).

---

## Workspace registry

Tags and macros resolve through a **workspace-effective registry** built at compile time. Nothing from a package enters the workspace until explicitly imported — same discipline as TypeScript `import { … } from` and Python `from … import`.

### Why workspace scope

| Problem with a flat global registry | Workspace fix |
| --- | --- |
| Two packages define `@compliance` | Selective `import` + optional `as` qualifier |
| Importing KYC pulls 40 unused tags | `import { only, needed } from @pkg;` |
| Transitive deps pollute consumer scope | Only **re-exported** registry symbols from dependencies |
| Core vs custom tags mixed | **Categories** + **tiers** |

### Tiers

| Tier | Source | In workspace how | Default category |
| --- | --- | --- | --- |
| **kernel** | pactiac builtin catalog | Always — no `import` | `core` |
| **stack** | `@stack { }` on `product` | Always when stack selected | `stack` |
| **std** | `@pactia/*` implied by stack / `pactia init` | Merged under stack precedence | `protocol`, `surface`, … |
| **package** | `import @scope/name` | **Only imported names** (or full export with `import * from`) | Author `export` on `define tag` / `define macro` |
| **local** | `define template` in product | Templates only — **not** new `@tag` / `#[macro]` names | — |

**Rule:** consumer products never register new global tag or macro names. Custom tags and macros are published in packages, then imported into the workspace.

### Categories

Every tag and macro in a package manifest carries a **category** for tooling, docs, and IDE grouping — not a separate namespace.

| Category | Typical members |
| --- | --- |
| `core` | Kernel tags: `@entity`, `@api`, `@auth`, `@guide`, … |
| `stack` | Stack macros: `#[database]`, platform defaults |
| `protocol` | `@api`, REST/GraphQL/gRPC wire tags and macros |
| `surface` | `@web`, `@ios`, `#[form]`, `#[a11y]` |
| `compliance` | HIPAA, GDPR, sanctions tags |
| `domain` | Vertical domain tags (`@escrow`, `@listing`, …) |
| `extension` | Org-specific custom tags (default for package authors) |

Package authors set category on registry definitions:

```pactia
define tag sanctions_check {
  category compliance
  scope endpoint
  body { level: string, provider: string, }
  lowers { product.yaml security.sanctionsChecks[] }
}
```

Omitting `category` defaults to `extension`. Kernel entries are always `core`.

### Building `effectiveRegistry`

Per compile unit (single file or [workspace](language-spec.md#workspace-layout)):

```
1. Seed kernel tags[] (category core) — always visible
2. Load stack package from @stack { } → merge tags[] + macros[] (category stack; overrides std macros)
3. Merge implied @pactia/* std registry per stack profile (lowest macro precedence before builtins)
4. Walk `import` statements in scope chain for the current file:
     product.pactia imports → whole workspace
     module.pactia imports  → that module subtree
     service.pactia imports → that service subtree
5. For each `import`: look up package in `kabol.lock`; import symbols per Rust path rules
6. Apply `import … as alias` → qualified prefix for invocation
7. Reject REGISTRY_COLLISION on duplicate unqualified names in the same scope
8. Parse and expand source using effectiveRegistry at that file's scope
```

Transitive package dependencies (in `pactia.package.yaml` `dependencies:`) do **not** add tags/macros to the consumer workspace unless the direct dependency **re-exports** them in manifest `registry.reexports[]`.

### Import paths

Versions live in **`kabol.toml`** (ranges) and **`kabol.lock`** (pins) — never in `import`. See [packages.md — Dependencies vs import](packages.md#dependencies-vs-import-cargo-model).

```pactia
import @pactia/kyc-compliance;
import * from @pactia/kyc-compliance;
import { sanctions_check, sanctions_screen } from @acme/fintech-rules;
import compliance from @pactia/hipaa;
import @acme/fintech-rules as fintech;
import @pactia/hipaa as hipaa;
import { compliance, phi_screen } from @pactia/hipaa;
```

| Form | Meaning |
| --- | --- |
| `import @scope/name;` | Package prelude (default exported registry symbols) |
| `import * from @scope/name;` | All exported `registry.tags[]` + `registry.macros[]` |
| `import symbol from @scope/name;` | One tag or macro |
| `import { a, b } from @scope/name;` | Listed symbols only |
| `import @scope/name as alias;` | Package qualifier: `@alias::tag`, `#[alias::macro]` |
| `import symbol as alias from @scope/name;` | Rename one symbol |

`VERSION_IN_IMPORT` if an `import` line contains semver. `DEPENDENCY_NOT_DECLARED` if `kabol.toml` lacks the package.

Selective import does **not** merge package `domain` / `rules` IR unless those symbols are marked `export` in package source — registry import and AST export share the same `export` modifier.

### Invocation rules

| Situation | Syntax |
| --- | --- |
| Unique unqualified name in effectiveRegistry | `@sanctions_check { }`, `#[list]` |
| Name imported only under `as fintech` | `@fintech::sanctions_check { }`, `#[fintech::list]` |
| Same simple name from two packages | `REGISTRY_COLLISION` — use selective import with one unqualified, or qualify both |
| Symbol not imported | `TAG_UNKNOWN` / `MACRO_UNKNOWN` |
| Symbol not in package export list | `IMPORT_NOT_EXPORTED` |

### Registry errors (workspace)

| Code | Condition |
| --- | --- |
| `TAG_UNKNOWN` | `@name` not in kernel, stack, std, or any `import` in scope chain |
| `MACRO_UNKNOWN` | `#[name]` not in effectiveRegistry for this file |
| `IMPORT_NOT_EXPORTED` | `import symbol from @pkg` names a symbol not marked `export` in package source |
| `DEPENDENCY_NOT_DECLARED` | `import @scope/name` without `kabol.toml` entry |
| `VERSION_IN_IMPORT` | Semver in an `import` statement |
| `REGISTRY_COLLISION` | Two imports expose the same unqualified tag/macro name |
| `REGISTRY_QUALIFIER_REQUIRED` | Ambiguous name — compiler requires `@alias::name` |
| `DEFINE_TAG_IN_PRODUCT` | `define tag` in consumer product |
| `DEFINE_MACRO_IN_PRODUCT` | `define macro` in consumer product |

---

## Tags

## Syntax rules

Tags are **annotations on named clauses** inside a host block — see [Tag applications](language-spec.md#tag-applications-annotation-model). Syntax:

```
@tagName clauseName {
  key: value
  nested: { ... }
  > prose
}
```

Bare `@tag { }` without `clauseName` is allowed only for **singleton** tags per scope (`@topology`, `@guide` on product). Multi-instance tags (`@entity`, `@event`, `@config`, `@integration`, `@api`) **require** a target name.

Tags attach to the **nearest host scope** and **cascade downward** unless overridden — see [Tag cascade](#tag-cascade-inheritance).

---

## Tag scope matrix

Canonical shapes match [fleet-management-v2.pactia](../fixtures/kernel/fleet-management-v2.pactia). **Role:** `clause` declares; `modifier` prefixes a host; `macro` expands at compile time. **Body style:** `assignments` (comma fields), `map` (named sub-clauses as keys), `entity-fields` (`name: Type,`), `flag` (no body), `shorthand` (single ref/number).

| Tag | Role | Allowed hosts | Target | Body style | Lowers to |
| --- | --- | --- | --- | --- | --- |
| `@stack` | clause | `product` | required | assignments | `product.stackId` |
| `@topology` | clause | `product` | omit | assignments | `product.topology` |
| `@tenancy` | clause | `product` | omit | assignments | `product.tenancy` |
| `@guide` | clause | `product`, `module`, `service` | optional | prose | product / module / service YAML |
| `@actor` | clause | `module` | required | assignments | `modules/<module>/<module>.module.yaml` `actors[]` |
| `@rule` | clause | `module`, `model` | required | prose or assignments | `<module>.module.yaml` or `<module>.model.yaml` `rules[]` |
| `@config` | clause | `module` | required | map | `modules/<module>/<module>.module.yaml` `config` |
| `@errors` | clause | `module` | required | map | `modules/<module>/<module>.module.yaml` `errors.catalog` |
| `@throws` | modifier | `@api` | omit | `{ names: [...] }` | `endpoint.errors` — references catalog |
| `@event` | clause | `module` | reference | assignments | `modules/<module>/<module>.module.yaml` `events[]` |
| `@entity` | clause | `model` | PascalCase | entity-fields | `modules/<module>/<module>.model.yaml` |
| `@enum` | clause | `model` | PascalCase | assignments | `modules/<module>/<module>.model.yaml` |
| `@relation` | clause | `model` | snake_case | assignments | `modules/<module>/<module>.model.yaml` |
| `@states` | clause | `model` | snake_case | assignments + `transitions: []` | `modules/<module>/<module>.model.yaml` |
| `@integration` | clause | `module` | snake_case | assignments | `modules/<module>/<module>.module.yaml` `integrations[]` |
| `@observe` | clause | `module` | snake_case | `slos: []` | `<module>.module.yaml` |
| `@deploy` | clause | `module` | required | nested clauses | `product.yaml` (`deployment`) |
| `@environment` | clause | `@deploy` | required | assignments | `product.yaml` `deployment.environments[]` |
| `@gate` | clause | `@deploy` | required | assignments | `product.yaml` `deployment.gates[]` |
| `@security` | clause | `module` | required | prose / assignments | `product.yaml` (`security`) |
| `@policy` | clause | `module` | required | assignments | `product.yaml` (`security`) |
| `@api` | clause | `service`, template | snake_case | assignments | `modules/<module>/services/*.service.yaml` |
| `@web` / `@ios` | clause | `@api` | snake_case | assignments | `product.yaml` (`surfaces`) |
| `@test` | clause | `service` | snake_case | assignments | service YAML `scenarios[]` |
| `@must` | clause | `service` | snake_case | assignments + prose | service YAML `obligations[]` |
| `@auth` | modifier | `@api` | omit | assignments | `authorization` |
| `@public` | modifier flag | `@api` | omit | flag | `authorization.type: PUBLIC` |
| `@body` | modifier shorthand | `@api` | omit | type ref | `request.dto` |
| `@returns` | modifier shorthand | `@api` | omit | type ref | `response.dto` |
| `@status` | modifier shorthand | `@api` | omit | number | `response.status` |
| `@emit` | modifier shorthand | `@api` | omit | event ref | `emits` |
| `@pk` `@unique` `@index` `@nullable` `@pii` | modifier flag | `@entity` field | omit | flag | field annotations |
| `@fk` | modifier | `@entity` field | omit | `{ entity: T }` | `annotations.references` |
| `@bind` | modifier | `@web`, `@ios` | omit | assignments | optional — inherits from `@api` |
| `#[database]` etc. | macro | `service` | — | expands | `service.flags` |
| `#[list]` `#[owner]` etc. | macro | `@api` | — | expands | endpoint patterns |

JSON Schemas: `spec/schemas/tags/<name>-v1.json` (planned) — IDE completion keys off the same schemas `pactiac` validates at compile time.

---

## Core tags (compiler built-in)

### Domain modeling (kernel — required in `model { }`)

| Tag | Example | Lowers to |
| --- | --- | --- |
| `@entity` | `@entity Vehicle { @pk id: uuid, ... }` | `modules/<module>/<module>.model.yaml` entities + fields |
| `@enum` | `@enum VehicleStatus { values: [ACTIVE, INACTIVE], }` | `<module>.model.yaml` enums |
| `@relation` | `@relation customer_owns { from: Customer, to: Vehicle, verb: owns, cardinality: many, }` | `<module>.model.yaml` relations |
| `@states` | `@states vehicle_lifecycle { entity: Vehicle.status, transitions: [{ from: ACTIVE, to: INACTIVE }], }` | `<module>.model.yaml` state machines |

Field modifiers **prefix** the field line (`@pk`, `@fk { entity: Customer }`) — not inline after the type.

### Roles

| Tag | Example | Lowers to |
| --- | --- | --- |
| `@actor` | `@actor customers { role: Customer, capabilities: [place_orders, view_orders], }` | `<module>.module.yaml` `actors[]` |

### Product / platform

| Tag | Example | Lowers to |
| --- | --- | --- |
| `@stack` | `@stack rust-anb { }` | `product.stackId` (version from `kabol.toml` / `kabol.lock`) |
| `@topology` | `@topology { mode: microservices, }` | `product.topology` |
| `@tenancy` | `@tenancy { mode: single, }` | `product.tenancy` |

### Service options (tags, not macros)

Prefer macros `#[database]` `#[cache]` `#[events]` **immediately above** `service Name { }` — they expand to `@database { }`, etc. See [language-spec.md — Prefix decorations](language-spec.md#prefix-decorations-macros-and-modifier-tags).

| Tag | Example | Lowers to |
| --- | --- | --- |
| `@database` | `@database { }` | `service.database: true` |
| `@cache` | `@cache { }` | `service.cache: true` |
| `@events` | `@events { }` | `service.events: true` |

### HTTP / API (tags only — modifiers are macros)

| Tag | Example | Lowers to |
| --- | --- | --- |
| `@auth` | `@auth { roles: [Customer, Admin] }` | `authorization.roles` |
| `@public` | `@public` | `authorization.type: PUBLIC` |
| `@body` | `@body CreateVehicleRequest` | `request.dto` |
| `@returns` | `@returns VehicleListResponse` | `response.dto` |
| `@status` | `@status 201` | `response.status` |
| `@emit` | `@emit vehicle.created` | `emits` |
| `@throws` | `@throws { names: [NotFound, Forbidden] }` on `@api` | `response.errors` |
| `@transition` | `@transition { from: PENDING, to: PAID }` | statemachine edge on endpoint |

Endpoint **patterns** use macros — not tags: `#[list]` `#[paginated]` `#[owner]` `#[create]` `#[idempotent]` — see #macros.

### Data field tags

Prefix modifier tags on the line **above** the field declaration:

| Tag | Example | Lowers to |
| --- | --- | --- |
| `@pk` | `@pk` then `id: uuid` | `annotations.primary` |
| `@fk` | `@fk { entity: Customer }` then `customerId: uuid` | `annotations.references` |
| `@unique` | `@unique` then `vin: string` | `annotations.unique` |
| `@pii` | `@pii` then `email: string` | `annotations.pii` |
| `@secret` | config / env only | `sensitive: true` |
| `@index` | `@index` then `customerId: uuid` | index hint in IR |
| `@nullable` | `@nullable` then `nextCursor: string` | `required: false` |
| `@retain` | `@retain { 7y }` on field | `policies.retention` |
| `@encrypt` | `@encrypt { at_rest }` | `policies.encryption` |

### Contract blocks

| Tag | Example | Lowers to |
| --- | --- | --- |
| `@config` | `@config backend { DATABASE_URL: { required: true, ... }, }` | `<module>.module.yaml` `config` |
| `@errors` | `@errors platform { NotFound: { status: 404, code: RESOURCE_NOT_FOUND, message: "..." }, }` | `<module>.module.yaml` error catalog |
| `@policy` | `@policy { retain Entity forever because "..." }` | `product.yaml` (`security`) |
| `@guide` | `@guide { Handlers use error envelope }` | module / service / product YAML (not enforced) |
| `@security` | `@security { ... }` | `product.yaml` (`security`) |
| `@compliance` | `@compliance { gdpr ... }` | `product.yaml` (`security`) + model field annotations |
| `@observe` | `@observe fleet_slos { slos: [...] }` | `<module>.module.yaml` |
| `@deploy` | `@deploy fleet { @environment staging { replicas: 2 }, ... }` | `product.yaml` (`deployment`) |
| `@event` | `@event { vehicle.created payload X handler Service.method "desc" }` | `<module>.module.yaml` `events[]`, `eventHandlers[]` |
| `@test` | `@test { "name" When ... Then ... }` | service YAML `scenarios[]` |
| `@must` | `@must { on trigger outcome lines }` | service YAML `obligations[]` |
| `@rule` | `@rule single_customer { > Vehicles belong to exactly one customer. }` | `rules[]` (enforced) |

See [Cross-cutting concerns](#cross-cutting-concerns) for cascade rules.

### Wiring and integration

| Tag | Example | Lowers to |
| --- | --- | --- |
| `@bind` | `@bind { service: FleetService, method: GET, path: "/api/v1/vehicles" }` | cross-link in `product.yaml` (`surfaces`) |
| `@bind` | `@bind { data: VehicleListResponse }` | data binding for UI |
| `@integration` | `@integration gps_devices { direction: inbound, maps_to: "POST /path", ... }` | `<module>.module.yaml` `integrations[]` |

`@integration` — target names the integration; body holds properties:

| Property | Required | Values |
| --- | --- | --- |
| *(target e.g. `gpsDevices`)* | yes | integration id — `integrations[].name` |
| `direction` | yes | `inbound` \| `outbound` |
| `auth` | yes | `{ type: api_key, env: VAR }` \| `{ type: hmac, header: NAME }` |
| `maps_to` | yes when inbound | string `"METHOD /path"` |
| `> sentence` | no | `integrations[].purpose` |

Full lowering table: [language-spec.md — @integration](language-spec.md#example--integration-kernel-tag).

Event **declarations** and **handler wiring** both belong in `@event { }`:

```pactia
@event {
  vehicle.created payload VehicleCreatedPayload
  handler NotificationService.onVehicleCreated
  > Fired when a vehicle is registered
}
```

A line like `> on vehicle.created -> NotificationService.onVehicleCreated` **outside** `@event { }` is prose — not lowered to `eventHandlers[]`.

---

## Protocol tags (package-registered — required for APIs)

| Tag | Package | Example |
| --- | --- | --- |
| `@api` | `@pactia/protocol-rest` | `@api { method GET path /api/v1/vehicles @auth { Customer } ... }` |
| `@grpc` | `@pactia/protocol-grpc` | `@grpc { service trade.TradeService rpc MarkPaymentSent }` |
| `@graphql` | `@pactia/protocol-graphql` | `@graphql { type Query field vehicles }` |

REST endpoints **must** use `@api { }` after `import @pactia/protocol-rest`. The kernel does not parse bare `GET` / `POST` lines.

See [platform.md](platform.md#protocol-packages).

---

## Surface tags (package-registered)

Surface intent uses **block tags** for each platform. Nested tags and macros live inside:

```pactia
@web {
  @screen { vehicle-list }
  @route { /fleet/vehicles }
  #[a11y(WCAG-AA)]
  @bind { FleetService GET /api/v1/vehicles }
  > Customer browses their vehicles in a paginated table
}
```

| Tag | Package | Purpose |
| --- | --- | --- |
| `@web` | `@pactia/surface-react` | Web client block |
| `@ios` | `@pactia/surface-swiftui` | iOS block |
| `@android` | `@pactia/surface-compose` | Android block |
| `@desktop` | `@pactia/surface-electron` | Desktop block |
| `@screen` | surface packages | `@screen { id }` |
| `@route` | `@web` | `@route { /path }` |
| `@nav` | mobile packages | `@nav { TabName }` |

---

## Prose vs enforced tags

| Form | Treatment |
| --- | --- |
| `> sentence` (required prefix) | `prose[]` — AI context, not enforced alone |
| `@rule { sentence }` | `rules[]` — enforced |
| `@actor { role: capabilities: }` | `actors[]` |
| `@test { ... }` | `scenarios[]` |
| `@must { on trigger ... }` | `obligations[]` |

---

## Package tags (`define tag`)

Package authors register new `@name { }` tags in **`index.pactia`** with `define tag` — normative grammar: [packages.md](packages.md#package-authoring); overview: [language-spec.md](language-spec.md#define-tag--package-registry-not-in-consumer-products) and [packages.md](packages.md#package-registry).

```pactia
define tag compliance {
  category compliance
  scope module
  body {
    framework: string,
    baa_required: boolean,
  }
  lowers {
    product.yaml security.sanctionsChecks[]
  }
}
```

At `pactia package build` → `registry.tags[]` + `schemas/compliance-v1.json` in the published tarball.

Consumers (selective import — recommended):

```pactia
import compliance from @pactia/hipaa;

@compliance {
  framework: hipaa,
  baa_required: true,
}
```

Or with crate alias when combining multiple compliance packages:

```pactia
import @pactia/hipaa as hipaa;

@hipaa::compliance {
  framework: hipaa,
  baa_required: true,
}
```

`@compliance` is **unknown** without `import` bringing it into the workspace effectiveRegistry. With import, the compiler validates the body and routes per `lowers { }` with provenance `PACKAGE`.

Hand-authored `extensions[]` / `tags[]` in manifest remains valid (protocol packages, legacy stacks). **`define tag` is the Pactia-native authoring form**; YAML is the compiled interchange.

---

## Adding a new core tag

1. Propose lowering rule (which YAML path).
2. Core tags: Pactia RFC + update this registry.
3. Package tags: `define tag` in `index.pactia` **or** `tags[]` in manifest — [packages.md](packages.md#package-registry).
4. Never add a kernel **keyword** — add a **tag** (`define tag` in a package) or register a **macro** (`define macro`).

---

## See also

- #macros — `#[list]` `#[owner]` and stack expansions
- [language-spec.md](language-spec.md)
- [Cross-cutting concerns](#cross-cutting-concerns)
- [platform.md](platform.md#protocol-packages)
- [packages.md](packages.md)

---

## Macros

## Tags vs macros

| Syntax | Name | Behavior |
| --- | --- | --- |
| `@tag { ... }` | **Tag** | Structured fact — compiled verbatim into IR |
| `#[macro]` / `#[macro(a, b)]` | **Macro** | Pattern — expanded from stack, domain, or protocol package definition |

**Boundary rule:** if it needs `{ }` to express its content, it is a **tag**. If it fits on one line with zero or scalar arguments, it is a **macro**.

---

## Syntax

```
#[list]
#[paginated]
#[owner]
#[rate_limit(100, rpm)]
#[retry(3, exponential)]
#[cache(ttl: 300)]
```

Macros attach to the **nearest `@api { }` block**, `service` block, or surface block (`@web { }`, …). Multiple macros on one line are expanded left-to-right.

Arguments are **scalars only** — identifiers, numbers, strings, simple `key: value` pairs. No nested blocks inside macro argument lists.

---

## Expansion pipeline

```
1. Parse source (tags, macros, facts, prose)
2. Resolve `@stack { }` + `import` packages; load macros[] + tags[] into effectiveRegistry
3. Expand define (templates only) in product source
4. Expand all #[macro] using effectiveRegistry (deterministic, pure)
5. Validate @tag { } bodies (kernel + package JSON schemas)
6. Lower expanded @tags → IR
7. Validate @pactia/schema + conformance
```

Every expanded field carries provenance `MACRO` pointing to the source macro name and package version. Conformance checks the **expanded** form.

| Code | Condition |
| --- | --- |
| `MACRO_UNKNOWN` | `#[name]` not registered by stack or any imported package |
| `MACRO_ARGS_INVALID` | Argument count or types fail package schema |
| `MACRO_EXPANSION_CYCLE` | Macro expansion references itself |

---

## Core macros (built-in, overridable by stack)

Default expansions below; stack packages **replace** these in `pactia.package.yaml` under `macros[]`.

### HTTP / API response shape

| Macro | Default expansion (conceptual) | IR effect |
| --- | --- | --- |
| `#[list]` | List response wrapper + default cursor from stack | `modifiers.list` |
| `#[paginated]` | `@response_includes { nextCursor }` + stack page defaults | `modifiers.paginated` |
| `#[detail]` | Single-entity detail response inference | `modifiers.detail` |
| `#[history]` | Child-entity collection response | `modifiers.history` |
| `#[create]` | Create request/response inference from entity | `modifiers.create` |

### Authorization / ownership (party scope)

| Macro | Default expansion | IR effect |
| --- | --- | --- |
| `#[owner]` | Filter rows where actor owns resource (FK inference) | `ownership.scope: OWN_ROWS` |
| `#[buyer]` | `auth.sub == resource.buyerId` | `ownership.scope: PARTY_BUYER` |
| `#[seller]` | `auth.sub == resource.sellerId` | `ownership.scope: PARTY_SELLER` |
| `#[participant]` | buyer OR seller on resource | `ownership.scope: PARTY_PARTICIPANT` |

### Endpoint behavior

| Macro | Default expansion | IR effect |
| --- | --- | --- |
| `#[idempotent]` | Require idempotency key | `idempotency: REQUIRED` |
| `#[rate_limit(n, unit)]` | Rate limit policy on endpoint | `endpoint.rateLimit` |

### Service flags

| Macro | Default expansion | IR effect |
| --- | --- | --- |
| `#[database]` | `@database { }` | `service.database: true` |
| `#[cache]` | `@cache { }` | `service.cache: true` |
| `#[events]` | `@events { }` | `service.events: true` |

---

## Registry precedence

When multiple packages register the same macro name, **the first winning layer** applies:

```
1. Stack package (`@stack rust-anb { }`)     ← highest
2. Explicit `import @scope/name` packages       ← in source order; collision → REGISTRY_COLLISION
3. @pactia/* standard library packages     ← e.g. @pactia/api-patterns
4. pactiac built-in default expansions     ← lowest
```

Tags: **kernel tags** are always available (`core`). **Package tags** exist in the workspace only after `import` (selective or wildcard). Two imports exposing the same unqualified name → `REGISTRY_COLLISION` unless one is qualified with `as`.

See [Workspace registry](#workspace-registry) for selective import and categories.

---

## Authoring macros — `define macro` or manifest

Package authors may register macros in **Pactia source** or **YAML manifest**. Build merges both.

### Pactia source (preferred)

```pactia
define macro list {
  expands {
    @cursor { default 20 max 100 }
    #[paginated]
  }
}
```

### Manifest (alternative or generated)

Stack authors may still use `yaml package/macros` or `macros[]` in `pactia.package.yaml`:

```yaml
macros:
  - name: list
    version: 1
    expands_to:
      - "@cursor { default 20 max 100 }"
      - "#[paginated]"
  - name: paginated
    version: 1
    expands_to:
      - "@response_includes { nextCursor }"
  - name: owner
    version: 1
    expands_to:
      - "@filter { actor.id == entity.ownerId }"
```

When `@pactia/rust-anb` changes pagination defaults, products recompile — **no `.pactia` source edits**.

**Product authors:** use `define template` for local block repetition — not `define macro`. **Package authors:** use `define macro` or manifest `macros[]`. See [language-spec.md](language-spec.md#define-macro--package-registry-not-in-consumer-products).

Domain and protocol packages may register additional macros the same way. Prefer **import @scope/name as alias`** or selective import when combining multiple packages — see [Workspace registry](registry.md#workspace-registry).

---

## Surface macros (package-registered)

| Macro | Package | Purpose |
| --- | --- | --- |
| `#[a11y(WCAG-AA)]` | surface packages | Accessibility standard on screen |
| `#[form]` | surface packages | Form layout pattern |
| `#[map]` | surface packages | Map view pattern |

---

## What is not a macro

These remain **tags** (always `{ }`):

- `@auth { roles: [Customer, Admin] }` — roles are facts
- `@returns VehicleListResponse` — response DTO is a fact
- `@body CreateVehicleRequest` — request DTO is a fact
- `@throws { names: [NotFound, Forbidden] }` — error list is a fact
- `@emit vehicle.created` — event name is a fact
- `@public` — public route is a fact
- `@transition { from: PENDING, to: PAID }` — legal edge is a fact
- `@stack rust-anb { }` — stack selection is a fact (version in `kabol.toml`)

---

## See also

- #tags — all `@tags`
- [platform.md](platform.md#stack-packages) — stack-owned macro tables
- [compilation.md](compilation.md) — expansion phase
- [packages.md](packages.md#extensibility) — `define` vs `#[macro]` decision table

---

## Cross-cutting concerns

## Three tiers

| Tier | Mechanism | Provenance | Enforced by conform? |
| --- | --- | --- | --- |
| **Platform law** | `@stack` + stack package | `STACK_DEFAULT` | Yes (tech policy) |
| **Product law** | `@policy { }`, `@must { }`, `@test { }`, field `@pii { }`, `@security { }` facts | `Pactia` | Yes |
| **Guidance** | `@guide`, `>`, free prose | `GUIDANCE` | No — AI + human review |

**Rule:** if it must never break in CI, express it as a **tag with schema** or link it to **`@test`**. If it is how AI should implement, use **`@guide`** or prose.

---

## Tag cascade (inheritance)

Tags and prose **attach to the nearest scope** and **cascade downward** unless overridden.

```
product
  └── module
        └── service
              └── `@api { }` / model field / `@web` screen
```

| Scope | Typical cross-cutting content |
| --- | --- |
| `product` | `@stack`, global `@guide`, import @acme/standards` |
| `module` | `@policy`, `@compliance`, module `>` rules |
| `service` | `@observe` SLOs, service `@guide`, `@security` rate limits |
| Endpoint | `#[rate_limit(n, unit)]`, endpoint-specific `@guide { }` prose |
| `model` field | `@pii { }`, `@retain { }`, `@encrypt { }` |
| Surface block | `#[a11y(WCAG-AA)]`, `@locale { }` |

Child scopes **merge** parent guidance into IR with `inheritedFrom` metadata. Overrides at lower scope win on conflict.

---

## Block tags (no new keywords)

Block tags use `@name { ... }`. Body is **`> prose` lines** (where narrative) and **tag-schema lines** / **nested `@tags`**.

### `@policy` — retention, residency, encryption (law)

```pactia
@policy {
  retain GpsPosition forever because "regulatory audit trail"
  retain Customer 7y after account closure
  residency EU
  encrypt PII at rest
}
```

| Lowers to | `product.yaml` (`security`) |
| Enforced | Yes |
| Provenance | `Pactia` |

Field-level override:

```pactia
email: string @pii { } @retain { 7y }
```

---

### `@guide` — best practices and coding standards (guidance)

```pactia
@guide {
  > All services follow clean architecture: api → application → domain → infrastructure
  > Handlers must map errors to the platform envelope before returning
  > Every mutating endpoint must be idempotent or document why not
  > Never log JWT tokens, secrets, or fields marked @pii
}
```

| Lowers to | Respective `.module.yaml` / `.service.yaml` (BSC may render agent briefs) |
| Enforced | No |
| Provenance | `GUIDANCE` |

**Shareable prompts:** org packages like import @acme/engineering-standards;` inject `@guide` blocks at publish time.

Place `@guide` at `product` (global), `module` (domain conventions), or `service` (local patterns).

---

### `@security` — security policy (law + guidance mix)

```pactia
@security {
  All admin actions must be audit-logged
  #[rate_limit(100, rpm)] on POST /api/v1/gps/ingest
  @require_mfa { Admin } in production
  @headers { HSTS max-age=31536000 } on all public endpoints
}
```

| Line type | Treatment |
| --- | --- |
| Prose sentence | `security.rules[]` — checked via `@test` when linked |
| `@rate_limit`, `@require_mfa`, `@headers` | Structured `security.controls[]` (tags) or macro expansion for `#[rate_limit(...)]` |

| Lowers to | `product.yaml` (`security`) |
| Enforced | Partial — structured tags yes; prose via linked `@test` |

---

### `@compliance` — vertical/regulatory packs (law)

Used with compliance packages (`import @pactia/gdpr-eu as gdpr`, `import compliance from @pactia/hipaa`).

```pactia
@compliance gdpr {
  applies to module fleet
  Customer email is personal data with lawful basis contract
  data subject erasure within 30 days on verified request
  @retain Customer 7y after account closure
}
```

Package registers `@compliance` block schema in `pactia.package.yaml` (same mechanism as protocol `@api`).

| Lowers to | `product.yaml` (`security`) + `<module>.model.yaml` field annotations + `<module>.module.yaml` `rules[]` |
| Enforced | Yes (schema-validated) |
| Provenance | `PACKAGE` + `Pactia` |

---

### `@observe` — SLOs, alerts, metrics (law when SLO conform exists)

```pactia
@observe {
  slo FleetService latency p99 < 300ms
  slo FleetService error_rate < 0.5%
  slo FleetService availability 99.9%

  alert high_error_rate on FleetService
    when error_rate > 2% for 5m
    severity critical
    notify on-call

  metric business_vehicles_active type gauge from Vehicle where status ACTIVE
}
```

Stack package supplies defaults (trace sampling, `/metrics` path). `@observe` only declares **product-specific overrides**.

| Lowers to | `<module>.module.yaml` |
| Enforced | Yes (when observability conform is enabled) |
| Provenance | `Pactia` or `STACK_DEFAULT` for derived metrics from `@emit` |

---

### `@deploy` — environments, replicas, promotion gates (law)

```pactia
@deploy fleet {
  @environment staging {
    replicas: 2,
    region: "eu-west-1",
  },
  @environment production {
    replicas: 3,
    region: "eu-west-1",
  },

  @gate production {
    scenarios: pass,
    coverage: ">= 80%",
  },
}
```

CI/CD **tool choice** (GitHub Actions, ArgoCD) stays in the **stack package**. `@deploy` only declares **product overrides** — environments, replica counts, promotion gates.

| Lowers to | `product.yaml` (`deployment`) |
| Enforced | Yes (infra / pipeline conform) |
| Provenance | `Pactia` |

---

## Inline tags and macros on endpoints and fields

| Form | Example | IR |
| --- | --- | --- |
| `#[rate_limit(n, unit)]` | `#[rate_limit(100, rpm)]` inside `@api { }` or `@security { }` | `endpoint.rateLimit` |
| `@require_mfa { }` | `@require_mfa { Admin }` | `security.mfa.roles` |
| `#[a11y(...)]` | `#[a11y(WCAG-AA)]` inside `@web { }` | `product.yaml` (`surfaces`) |
| `@retain { }` | `@retain { 7y }` on field | `policies.retention` |
| `@encrypt { }` | `@encrypt { at_rest }` on field | `policies.encryption` |
| `@audit { }` | `@audit { }` on `@api { }` POST | `security.auditRequired` |

---

## Prose forms (no block tag)

| Form | Use | IR |
| --- | --- | --- |
| `> sentence` | Business rule or narrative (guidance) | `prose[]` |
| Plain sentence in module | Constraint / MVP scope | **Invalid** — use `> sentence` |
| `@actor { role: capabilities: }` | Capabilities | `actors[]` |
| `@test { name: when: then: }` | Acceptance / policy checks | service YAML `scenarios[]` |

Example — MVP constraint as prose (not a keyword):

```pactia
> Single-region deployment for MVP — no multi-region failover required
```

---

## What goes where (cheat sheet)

| Concern | Expression |
| --- | --- |
| Language, framework, forbidden crates | `@stack` package — do not repeat |
| Handler error mapping style | Stack `codingStandards` or `@guide` |
| GPS history forever | `@policy { retain GpsPosition forever ... }` |
| EU residency | `@policy { residency EU }` |
| Email is PII | `email: string @pii { }` in `model` |
| 80% coverage before prod | `@gate production { coverage: ">= 80%" }` inside `@deploy` |
| p99 latency target | `@observe { slo ... }` |
| HIPAA PHI tagging | import @pactia/hipaa` + `@compliance` |
| Hexagonal architecture preference | `@guide` prose |
| Endpoint rate limit | `#[rate_limit(100, rpm)]` inside `@api { }` or `@security { }` |
| VoiceOver on iOS | `@ios { ... #[a11y(VoiceOver)] ... }` |
| Never log secrets | `@guide` or `@must always` + `@test` |

---

## Policy packages (shareable standards)

Publish org or community standards as pactia.io packages:

```pactia
import @pactia/gdpr-eu as gdpr;
import { compliance } from @pactia/gdpr-eu;
import * from @pactia/security-baseline;
import @acme/engineering-standards as standards;
```

| Package kind | Injects |
| --- | --- |
| `stack` | Platform law, CI baseline, coding standards |
| `domain` | Entities, rules, integrations |
| `policy` (new kind) | `@guide`, `@security`, `@compliance` blocks |
| `surface` | `@web { }`, `@ios { }` block schemas |

Consumer `product` inherits package guidance; overrides only where the product diverges.

---

## IR output files (module-scoped)

| Source | Output file |
| --- | --- |
| `@policy`, `@retain` on fields | `product.yaml` (`security`) |
| `@guide`, `>` rules (guidance only) | module / service / product YAML |
| `@security` | `product.yaml` (`security`) |
| `@compliance` | `product.yaml` (`security`) + `<module>.model.yaml` annotations |
| `@observe` | `<module>.module.yaml` |
| `@deploy` | `product.yaml` (`deployment`) |
| `@must`, `@test` | `modules/<module>/services/<service>.service.yaml` |
| Stack package | Merged at compile — not re-authored |

`bsc compile-workspace` assembles agent briefs from these IR slices for AI tools.

---

## Provenance summary

| Provenance | Meaning | Example |
| --- | --- | --- |
| `Pactia` | Author wrote enforceable fact | `@policy`, `@test`, `@auth` |
| `GUIDANCE` | Author wrote best practice | `@guide`, non-tested prose |
| `STACK_DEFAULT` | From stack package | Default retention, CI tool |
| `PACKAGE` | From `import` package | `@compliance gdpr` from `@pactia/gdpr-eu` |
| `INFERRED` | Compiler derived | Metric from `@emit` event name |
| `NOT_DERIVABLE` | Below the line hint | `implementation_hint` files |

Conformance **never** enforces `GUIDANCE` — only law tiers and linked `@test` assertions.

---

## Anti-patterns

| Do not | Do instead |
| --- | --- |
| Copy stack `codingStandards` into every product | import @stack` only |
| Put enforceable SLO only in `@guide` | `@observe { slo ... }` |
| `flow { step 1; step 2 }` for policy | `@must` + `@test` |
| New keyword `pipeline` | `@gate production { ... }` inside `@deploy` |
| Unstructured compliance in Slack | `@compliance` block + package |

---

## See also

- #tags — full `@tag` list
- [language-spec.md](language-spec.md)
- [platform.md](platform.md#stack-packages) — what stack owns vs product overrides
- [overview.md](overview.md#architecture-coverage) — architect concern matrix
- [overview.md](overview.md#the-intent-line) — law vs implementation
