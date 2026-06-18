# Pactia Language Specification

Version: **1.0**  
Status: **Specification**

Part of: [overview.md](overview.md) | [registry.md](registry.md) | [packages.md](packages.md) | [grammar-reference.md](grammar-reference.md)

---

## Three altitudes

Every Pactia file is legal at altitude 0. Tags are an opt-in upgrade, one fact at a time, never all-or-nothing.

> **The one rule that matters:** Pactia never requires more than prose. Every tag, every macro, every keyword beyond `pactia` + `product` is something you reach for when you want enforcement — never something you're forced to learn to get started.  
> — also stated in [overview.md](overview.md#three-altitudes)

| Altitude | What you write | When |
| --- | --- | --- |
| **0** | `> prose` in `product { }` — product description + optional agent rules | What the product is; coding standards for agents |
| **1** | `product` prose + light `@tag` | Product description in `product { }`; one fact per endpoint or block |
| **2** | Full tag + macro surface | Deterministic IR, conformance, multi-surface |

### Altitude 0 — prose only

```pactia
pactia 1.0

product MyApp {
  > A mobile app for tracking personal fitness goals and sharing progress with friends.
  > Never commit secrets. Map errors to our envelope before returning.
  > List endpoints use cursor pagination.
}
```

`>` prose in `product { }` with no `module`, `service`, or `data` — the smallest legal program. Include at least one line that says **what the product is**; additional lines are coding standards or agent rules.

### Altitude 1 — light tagging

```pactia
pactia 1.0

product MyApp {
  > A mobile app for tracking personal fitness goals and sharing progress with friends.

  module fitness {
    service WorkoutService {
      @api list_workouts {
        > Customers browse their workout history, paginated.
        @auth Customer
        @returns WorkoutListResponse
      }
    }
  }
}
```

Keep **what the product is** in `product { }` prose. Add `module`, `service`, and tags one at a time — endpoint prose inside `@api { }` describes that feature.

### Altitude 2 — fully specified

```pactia
@auth { roles: [Customer, Admin] }
#[list] #[paginated] #[owner]
@returns VehicleListResponse
@throws { names: [Forbidden] }
@api list_vehicles {
  method: GET,
  path: "/api/v1/vehicles",
}
```

Canonical reference: [fleet-management-v2.pactia](../fixtures/kernel/fleet-management-v2.pactia).

---

## What Pactia is

**Pactia is a shareable standard for AI-native product intent** — between natural language and code:

- Humans read it like a product spec.
- Any AI toolchain can consume compiled output — Pactia does not target a single model or IDE.
- `pactiac` lowers source to **AI-neutral YAML IR**; BSC renders per-consumer `specification/`.

You are writing the **permanent, versioned prompt** for your product. Teams publish `.pactia` files to pactia.io the same way they publish OpenAPI — except the spec covers **roles, data, APIs, rules, tests, UI intent, and stack** in one place.

---

## Design laws

1. **Small fixed keyword set** — structure, imports, and local template expansion (`define template`). Nothing else is reserved.
2. **Tags + macros + prose only** — every structured fact is `@tag { }`; every pattern is `#[macro]`; narrative context is **`> prose`**. **No kernel pattern matching** for HTTP verbs, roles, or relations.
3. **Still compilable** — tags lower verbatim; macros expand deterministically before IR emit.
4. **Multi-surface** — same file drives APIs and `@web { }` / `@ios { }` / `@android { }` / `@desktop { }` blocks.
5. **Shareable** — one file in git, one digest in `kabol.lock`, same context in every AI session.
6. **No behavior scripts** — no numbered flows; use `@must { }` for outcomes, `@test { }` for acceptance.
7. **Protocol-agnostic kernel** — REST, gRPC, GraphQL are **packages**. The kernel endpoint tag is **`@api`** — not protocol-flavored. Wire-format macros come from protocol packages.

---

## The nine kernel keywords

| Keyword | Purpose |
| --- | --- |
| `pactia` | Version line: `pactia 1.0` |
| `product` | Root block — the whole system |
| `module` | Capability group (bounded context) |
| `service` | Deployable unit for APIs and server-side logic (one surface of the product) |
| `data` | Entities, enums, relations, shapes |
| `import` | Dependency resolution: `import @pactia/kyc-compliance;` or `import "./data/vehicle.pactia"` — path only, no version |
| `export` | Symbol visibility: `export define tag …`, `export @entity …` — marks symbols consumers may import |
| `define` | Compile-time template or package registry (`define macro` / `define tag`) — expands to kernel constructs before IR emit |
| `yaml` | Escape hatch — raw YAML merge or package authoring |

Everything else is **prose**, **`@tag { }`**, or **`#[macro]`**.

English words such as `on`, `when`, `then`, `GET`, or `POST` may appear in **prose** or inside a **tag body**; the kernel does not treat them as syntax unless they appear inside `@tag { }`, `#[macro]`, or a `define template` body.

---

## Three line kinds (strict)

Every line in a `{ }` block is exactly one of:

| Kind | Syntax | Meaning |
| --- | --- | --- |
| **Tag** | `@name { ... }` | Structured fact — compiled verbatim into IR |
| **Macro** | `#[name]` or `#[name(args)]` | Registered pattern — expanded before IR emit (includes `define template` invocations) |
| **Prose** | `> sentence` | AI context — scoped to nearest block; not lowered as structured IR |

**No exceptions** — prose **always** begins with `>`, in `product { }` and everywhere else. There are no HTTP fact lines, no `Role can ...` lines outside `@actor { }`, no bare relation lines, and no bare `enum` / `Entity { }` declarations. Those facts live inside kernel tags: `@api { }`, `@actor { }`, `@relation { }`, `@entity { }`, `@enum { }`, `@states { }`.

**Boundary:** if it needs `{ }` for its content, it is a **tag**. If it is a one-line pattern reference, it is a **macro**. Narrative context uses **`>`**. See [registry.md](registry.md#tags) and [registry.md](registry.md#macros).

Lines that *look* like wiring but use `>` remain **guidance only** — e.g. `> on vehicle.created -> NotificationService.onVehicleCreated`. **Conformance** uses tags, expanded macros, `@rule { }`, `@test { }`, and `@must { }`.

---

## Prose and comments

### Prose (`>`)

Every **prose line** MUST start with `>` (after indentation). One rule everywhere — `product`, `module`, `service`, and inside tag bodies.

**Multiline prose** is multiple `>` lines in the same block. There is no block-prose syntax (no heredoc, no paragraph under a single `>`).

```pactia
product FleetManagement {
  > Platform for tracking vehicles and managing fleets

  @guide {
    > Handlers must map errors to the platform error envelope before returning
    > Never log JWT tokens or fields marked @pii
  }

  module fleet {
    > Single-region deployment for MVP — no real-time streaming required

    service FleetService {
      > Core fleet management and vehicle tracking

      @api {
        method GET
        path /api/v1/vehicles
        @web {
          @screen { vehicle-list }
          > Customer browses their vehicles in a paginated table
        }
      }
    }
  }
}
```

| Rule | Detail |
| --- | --- |
| **Prefix** | `>` required on every prose line — no bare sentences |
| **Multiline** | One `>` per line; same block scope accumulates all lines |
| Scope | Prose attaches to the nearest `{ }` block (`product`, `module`, `service`, `@api { }`, `@web { }`, …) |
| IR | Lowered to `prose[]` / `guidance.yaml` with provenance `GUIDANCE` — not conformance-enforced alone |
| Inside tag bodies | Schema lines do not use `>`; narrative uses `> sentence` — see [Unified tag body grammar](language-spec.md#unified-tag-body-grammar) |

| Error | Condition |
| --- | --- |
| `PROSE_PREFIX_REQUIRED` | Non-empty line looks like prose but does not start with `>` |
| `COMMENT_IN_IR` | Comment token appeared in lowered output (compiler bug) |

### Comments (`//` and `/* */`)

Comments are **not** part of the program. The lexer strips them before parse; they never appear in IR.

```pactia
// Package: @acme/fintech-rules — not visible to AI or conformance

product MyApp {
  > MVP scope only

  /* Multi-line note for authors:
     refactor modules after launch */

  module billing {  // TODO: split invoicing
    @rule { Invoices are immutable after issue }
  }
}
```

| Form | Rule |
| --- | --- |
| `// ...` | Line comment to end of line |
| `/* ... */` | Block comment; may span lines; does not nest |
| Inside strings | `"http://example.com"` — `//` inside quotes is literal |

Comments are for **authors and reviewers only**. AI agents must not treat comments as product law — use `>` prose or `@tag { }` for agent-visible intent.

---

## Tag system (overview)

Tags are the primary authoring surface in Pactia. Every structured fact uses `@tag`. Three **roles** — same symbol `@`, different placement and lowering:

| Role | Placement | Declares or decorates? | Example |
| --- | --- | --- | --- |
| **Clause tag** | Standalone line in host body | **Declares** a named clause | `@entity Vehicle { }`, `@api get_vehicle { }` |
| **Modifier tag** | Line **above** host (prefix) | **Decorates** next line | `@auth { }`, `@pk`, `@returns VehicleDetailResponse` |
| **Macro** | Line **above** host, or template invoke | **Expands** to tags before IR | `#[database]`, `#[list]` |

**Why tags vs macros:** tags map to **schema slots** in IR (validated JSON Schema per tag). Macros are **compile-time patterns** registered by stack/protocol packages — they expand to tags/macros deterministically.

Canonical reference file: [fleet-management-v2.pactia](../fixtures/kernel/fleet-management-v2.pactia). Runnable workspaces: [pactia-lang/examples](https://github.com/pactia-lang/examples). Full scope matrix: [registry.md — Tag scope matrix](registry.md#tag-scope-matrix).

---

## Tag applications (annotation model)

Clause tags work like **NestJS class/method decorators**: `@tagName clauseName { body }`.

| Part | Role | Example |
| --- | --- | --- |
| `@TagName` | Annotation **kind** | `@api`, `@entity` |
| `TagTarget` | **Clause id** (required for multi-instance) | `list_vehicles`, `Vehicle` |
| `{ ... }` | Structured body | `method: GET, path: "/api/v1/vehicles",` |

**Host blocks:** `product`, `module`, `service`, `data`, `@api { }`, `@web { }`, `@deploy { }`.

Full BNF: [grammar-reference.md](grammar-reference.md#tag-application-bnf).

### One form per modifier arity

Every modifier tag has exactly one canonical form:

| Arity | Form | Examples |
| --- | --- | --- |
| Zero | Flag | `@pk`, `@public`, `@unique` |
| One | Shorthand | `@returns VehicleListResponse`, `@status 201`, `@emit vehicle.created`, `@auth Customer` |
| Multiple | Body | `@auth { roles: [Customer, Admin] }`, `@fk { entity: Customer }` |

If a tag takes one value, the body form (`@returns { type: X }`) **does not parse**. There is no alternate spelling.

### Target naming conventions

| Kind | Convention | Valid | Invalid |
| --- | --- | --- | --- |
| REST / screen / test ids | `snake_case` | `get_vehicle`, `vehicle_list` | `get-vehicle` |
| Entities / DTOs | `PascalCase` | `Vehicle`, `CreateVehicleRequest` | `vehicle` |
| Events | `domain.action` reference | `vehicle.created` | `vehicle-created` |
| Module clauses | `snake_case` | `fleet_slos`, `gps_devices` | `fleetSlos` |
| Singleton tags on `product` | **omit target** | `@topology { mode: microservices }` | `@topology product { }` |

### When `TagTarget` is required

| Pattern | Target | Example |
| --- | --- | --- |
| **Multi-instance** | **Required** | `@entity Vehicle { }`, `@errors platform { }`, `@environment staging { }` |
| **Singleton** per host | **Omit** | `@topology { mode: microservices }` on `product` |
| **Modifier** on next line | **Omit** (host is implicit) | `@pk` then `id: uuid` |
| **Event** | **Reference** | `@event vehicle.created { }` |

### Modifier arity (summary)

See [One form per modifier arity](#one-form-per-modifier-arity). Full rules: [grammar-reference.md](grammar-reference.md#tag-application-bnf).

### Repeated fields → arrays

Object clauses must not repeat the same key. Use JSON-style arrays:

```pactia
@observe fleet_slos {
  slos: [
    { service: FleetService, metric: error_rate, target: "< 1%" },
    { service: FleetService, metric: availability, target: "99.9%" },
  ],
}

@states vehicle_lifecycle {
  entity: Vehicle.status,
  transitions: [
    { from: ACTIVE, to: INACTIVE },
    { from: ACTIVE, to: DECOMMISSIONED },
  ],
}
```

Duplicate key in clause body → `CLAUSE_DUPLICATE_KEY`.

### `@errors` vs `@throws`

| Tag | Role | Scope | Example |
| --- | --- | --- | --- |
| `@errors` | **Defines** catalog entries | `module` | `@errors platform { NotFound: { status: 404, ... } }` |
| `@throws` | **References** catalog on an API | prefix on `@api` | `@throws { names: [NotFound, Forbidden] }` |

Verb distance makes define vs reference obvious — `errors` declares, `throws` refers.

### `@bind` inheritance

`@web` / `@ios` clauses **inside** `@api` inherit `service`, `method`, and `path` from the parent `@api` when `@bind` is omitted:

```pactia
@api get_vehicle {
  method: GET,
  path: "/api/v1/vehicles/:id",

  @web vehicle_detail {
    @screen { id: vehicle_detail }
    @route { path: "/fleet/vehicles/:id" }
    > Customer opens vehicle detail
  }
}
```

Explicit `@bind { }` overrides inheritance for cross-service or alternate paths.

### Deprecated endpoint forms

`@api`, and legacy `{ type: X }` modifier wrappers still parse for backward compatibility. **New files should use `@api`** with prefix decorators. See [grammar-reference.md — Deprecated forms](grammar-reference.md#api-endpoints).

### Canonical example (excerpt)

See full file: [fleet-management-v2.pactia](../fixtures/kernel/fleet-management-v2.pactia).

```pactia
product FleetManagement {
  @topology { mode: microservices, }

  module fleet {
    @errors platform {
      NotFound: { status: 404, code: RESOURCE_NOT_FOUND, message: "..." },
    },

    data {
      @entity Vehicle {
        @pk
        id: uuid,
        @fk { entity: Customer }
        customerId: uuid,
      },
    }

    service FleetService {
      @auth { roles: [Customer, Admin] }
      @returns VehicleDetailResponse
      @throws { names: [NotFound, Forbidden] }
      @api get_vehicle {
        method: GET,
        path: "/api/v1/vehicles/:id",
      },
    }
  }
}
```

**Note:** clause bodies use **comma-separated fields**. See [Literals and clause fields](#literals-and-clause-fields).

### How lowering uses target + body

| Application | Target → IR key | Body → IR fields |
| --- | --- | --- |
| `@config backend { ... }` | `config.profiles.backend` | env var map |
| `@integration gps_devices { ... }` | `integrations[].id` | `direction`, `auth`, `maps_to` |
| `@entity Vehicle { ... }` | `domain.entities.Vehicle` | fields + field modifiers |
| `@errors platform { ... }` | `errors.catalog` | error definitions |
| `@throws { names: [...] }` on `@api` | `endpoint.errors` | catalog refs |
| `@api get_vehicle { ... }` | `services.*.endpoints[].id` | `method`, `path`, surfaces |

### Field-level modifiers (prefix)

```pactia
@entity Customer {
  @pii
  @unique
  email: string,
}
```

Inline modifiers after the type → `DECORATOR_MUST_PREFIX`.

### Author errors

Errors a beginner writing altitude 0–1 Pactia will realistically hit:

| Code | Condition |
| --- | --- |
| `TAG_UNKNOWN` | `@name` not in kernel or any imported package |
| `STRING_REQUIRED` | Path or message not wrapped in `"..."` |
| `PROSE_QUOTED` | Prose incorrectly wrapped in quotes instead of using `>` |
| `PROSE_PREFIX_REQUIRED` | Non-empty line looks like prose but does not start with `>` |
| `TAG_BODY_INVALID` | Tag body fails schema — wrong field or arity |

Implementer codes (registry collision, decorator placement, clause validation): [grammar-reference.md](grammar-reference.md#implementer-error-codes).

See [registry.md — Tag scope matrix](registry.md#tag-scope-matrix).

---

## Prefix decorations (macros and modifier tags)

Pactia follows **NestJS decorators** and **Rust attributes**: decorations sit **immediately above** the clause they modify.

**Rule:** one or more decoration lines prefix a single host line. Read order is **bottom-up**.

| Host | Decoration examples |
| --- | --- |
| `service Name { }` | `#[database]` `#[cache]` `#[events]` |
| Field line in `@entity { }` | `@pk`, `@fk { entity: Customer }`, `@pii` |
| `@api id { }` | `@auth { }`, `#[list]`, `@returns VehicleDto`, `@body CreateRequest`, `@status 201` |
| `@web id { }` / `@ios id { }` | `#[form]`, `#[a11y(WCAG_AA)]` |

Placement rules and implementer errors: [grammar-reference.md](grammar-reference.md#tag-application-bnf).

```pactia
#[database]
#[cache]
#[events]
service FleetService {
  @auth { roles: [Customer, Admin] }
  #[list] #[paginated] #[owner]
  @returns VehicleListResponse
  @throws { names: [Forbidden] }
  @api list {
    method: GET,
    path: "/api/v1/vehicles",
    @web vehicle_list {
      @screen { id: vehicle_list }
      @route { path: "/fleet/vehicles" }
      > Customer browses their vehicles in a paginated table
    },
  },
}
```

---

## Literals and clause fields

Pactia uses familiar value syntax — no bespoke string or object rules beyond `> prose` and Pactia-specific `@tag` / `#[macro]` forms.

### Strings

**All human text, paths, URLs, version constraints, and scenario steps must be double-quoted strings:**

```pactia
@rule single_customer {
  > Vehicles belong to exactly one customer.
}

path: "/api/v1/vehicles",
version: "^1.0",
when: "Admin is logged in and POST /api/v1/vehicles with valid body",
```

| Must be `"..."` | Examples |
| --- | --- |
| Paths and routes | `"/api/v1/vehicles"`, `"/fleet/vehicles/:id"` |
| Messages and statements | `"Payment could not be captured"` |
| `@test` steps | `when: "..."`, `then: "..."` |
| Constraints with symbols | `"^1.0"`, `">= 80%"`, `"< 1%"` |
| Regions / env-like tokens with `-` | `"eu-west-1"` |
| HTTP method + path composites | `maps_to: "POST /api/v1/gps/ingest"` |
| Array elements when the value is human text | `tags: ["beta", "internal-only"]` — rare; prefer identifiers in `[]` |

**Exception — prose:** a single narrative line starting with `>` is freeform text and is **never** quoted:

```pactia
> Customer browses their vehicles in a paginated table
```

### Non-string values

Unquoted values must follow normal programming-language naming rules:

```
Identifier ::= [A-Za-z_][A-Za-z0-9_]*
Reference  ::= Identifier ("." Identifier)+
Number     ::= [0-9]+
Boolean    ::= true | false
```

| Form | Examples |
| --- | --- |
| **Identifier** | `GET`, `POST`, `Customer`, `Admin`, `microservices`, `api_key`, `get_vehicle` |
| **Reference** | `Vehicle.status`, `vehicle.created`, `NotificationService.onVehicleCreated`, `OrderStatus.PAID` |
| **Number** | `201`, `404`, `1000` |
| **Boolean** | `true`, `false` |
| **Type name** (in `@entity`) | `uuid`, `string`, `VehicleListItem[]` |

### Numbers

Numeric literals are **bare integers** — never quoted. Use them wherever the schema slot is numeric:

| Slot | Example | Do not use |
| --- | --- | --- |
| HTTP status | `status: 404,` | `status: "404"` |
| Response status on `@api` | `@status 201` | `@status "201"` |
| Replicas | `replicas: 2,` | `replicas: "2"` |
| Rate limit | `#[rate_limit(1000, rpm)]` | `#[rate_limit("1000", rpm)]` |

```pactia
@errors platform {
  NotFound: {
    status: 404,
    code: RESOURCE_NOT_FOUND,
    message: "The requested resource does not exist",
  },
  Forbidden: {
    status: 403,
    code: ACCESS_DENIED,
    message: "Actor lacks permission",
  },
}

@status 201

@environment staging {
  replicas: 2,
}
```

Numbers lower to JSON numbers in IR. Quoting a numeric value (`"404"`) → `INVALID_LITERAL` when the schema expects a number.

### Array literals

Multi-value fields use **JSON-style array syntax** — square brackets with comma-separated elements:

```
ArrayLiteral  ::= "[" ArrayElement ("," ArrayElement)* (",")? "]"
ArrayElement  ::= String | Identifier | Reference | Number | Boolean
```

| Use | Example | Meaning |
| --- | --- | --- |
| **Value array** (tag field) | `roles: [Customer, Admin],` | field accepts one or more values |
| **Value array** (capabilities) | `capabilities: [manage_fleets, manage_users],` | each element is an Identifier (`snake_case`) |
| **Enum values** | `values: [ACTIVE, INACTIVE, DECOMMISSIONED],` | enum member identifiers |
| **Error names** | `names: [NotFound, Forbidden],` | catalog references |
| **Type array** (`@entity` only) | `vehicles: VehicleListItem[],` | field type is an array of `Type` — `[]` suffix on the type |

**Rules:**

- If a tag schema slot is **array-typed**, authors **must** use `[...]` — never repeat the same key on multiple lines.
- Array elements follow the same literal rules as scalars: identifiers/references unquoted, human text in `"..."`.
- Capability and role tokens are **identifiers** (`manage_fleets`), not quoted phrases (`"manage fleets"`).
- Trailing comma after the last element is allowed (same as object fields).

```pactia
@actor admins {
  role: Admin,
  capabilities: [manage_fleets, manage_users],
}

@auth { roles: [Customer, Admin] }

@throws { names: [NotFound, Forbidden] }

@enum VehicleStatus {
  values: [ACTIVE, INACTIVE, DECOMMISSIONED],
}
```

| Error | Condition |
| --- | --- |
| `ARRAY_REQUIRED` | Schema expects multiple values but author used a bare scalar |
| `ARRAY_ELEMENT_INVALID` | Element inside `[...]` breaks String / Identifier / Reference rules |
| `SCALAR_EXPECTED` | Schema expects one value but author used `[...]` |

Unquoted tokens that contain spaces, `/`, `:`, `-` (except inside identifiers), or other punctuation → `INVALID_LITERAL` at compile time.

Tag **targets** (`@api get_vehicle`, `@entity Vehicle`) use **Identifier** form — prefer `snake_case` or `PascalCase`, not kebab-case (`get-vehicle` is invalid).

### Comma-separated clause fields

Every object-shaped clause — tag bodies, `@entity` field lists, inline `{ }` blocks — separates members with **commas**, like Rust structs and TypeScript interfaces:

```pactia
@entity Vehicle {
  @pk { }
  id: uuid,
  @fk { entity: Customer }
  @index { }
  customerId: uuid,
  @unique { }
  vin: string,
  label: string,
  status: VehicleStatus,
  deviceId: string,
  createdAt: datetime,
}

@relation customerOwnsVehicles {
  from: Customer,
  to: Vehicle,
  verb: owns,
  cardinality: many,
}

@config backend {
  DATABASE_URL: {
    required: true,
    > PostgreSQL connection string for the fleet database.
  },
}

@api list_vehicles {
  method: GET,
  path: "/api/v1/vehicles",
}

@deploy fleet {
  @environment staging {
    replicas: 2,
    region: "eu-west-1",
  },
  @gate production {
    scenarios: pass,
    coverage: ">= 80%",
  },
}
```

| Rule | Detail |
| --- | --- |
| **Trailing comma** | Allowed after the last field (Rust-style) |
| **Prose lines** | `> ...` — no comma; may appear anywhere among fields |
| **Prefix decorations** | `@pk { }`, `#[list]` — on their own line above the field or host; not comma-separated |
| **Field line** | `name: Type,` — type is an Identifier or `Type[]` reference |

### Errors

| Code | Condition |
| --- | --- |
| `STRING_REQUIRED` | Value looks like text/path but is not wrapped in `"..."` |
| `INVALID_LITERAL` | Unquoted token breaks Identifier / Reference / Number rules |
| `CLAUSE_FIELD_SEPARATOR` | Object clause field not followed by `,` (or `}` / prose / decoration) |
| `PROSE_QUOTED` | Prose line incorrectly wrapped in quotes instead of using `>` |

---

## Unified tag body grammar

Every `@tag { }` block follows one rule: inside `{ }`, lines are comma-terminated fields/assignments **or** prose (`> ...`). Bare unquoted sentences inside tag bodies → `TAG_BODY_INVALID`.

Full BNF and schema line styles: [grammar-reference.md](grammar-reference.md#unified-tag-body-grammar).

**No clause exceptions** — `@actor`, `@test`, `@must`, `@relation`, `@states` use assignments (`role:`, `when:`, `from:`) or `> prose`, not bare sentence templates.

### Example — `@integration` (kernel tag)

**Authoring (canonical — target names the integration clause):**

```pactia
@integration gps_devices {
  direction: inbound,
  auth: {
    type: api_key,
    env: GPS_DEVICE_API_KEY,
  },
  maps_to: "POST /api/v1/gps/ingest",

  > Ingests GPS position updates from telematics devices
}
```

| Property | Required | Type / values |
| --- | --- | --- |
| *(target `gps_devices`)* | yes | integration id — `integrations[].name` |
| `direction` | yes | `inbound` \| `outbound` |
| `auth` | yes | object — `{ type, env }` or `{ type, header }` |
| `maps_to` | yes when `direction: inbound` | string `METHOD /path` — must match an `@api` endpoint |
| `> ...` | no | prose — lowers to `integrations[].purpose` |

**Lowering → `input/integrations.yaml`:**

```yaml
integrations:
  - name: gps_devices
    purpose: Ingests GPS position updates from telematics devices
    direction: INBOUND
    auth:
      type: API_KEY
      envVar: GPS_DEVICE_API_KEY
    mapsToEndpoint: POST /api/v1/gps/ingest
    timeout: 5s          # STACK_DEFAULT when omitted
    retry: 3 times with exponential backoff
```

| Source | IR field | Provenance |
| --- | --- | --- |
| `@integration gps_devices { ... }` | `name` from **target** | `Pactia` |
| `> Ingests GPS...` | `purpose` | `GUIDANCE` |
| `direction: inbound` | `direction: INBOUND` | `Pactia` |
| `auth: { type: api_key env: ... }` | `auth.type`, `auth.envVar` | `Pactia` |
| `maps_to: POST /path` | `mapsToEndpoint` | `Pactia` |
| `requestBody` / `responseBody` | Inferred from `maps_to` endpoint `@body` / `@returns` | `INFERRED` |

**Anti-patterns:**

```pactia
// BAD — legacy space slots (warn LEGACY_SLOT_SYNTAX); use key: value
@integration {
  inbound api_key GPS_DEVICE_API_KEY
  > GpsDeviceProvider ingests GPS updates
  maps to POST /api/v1/gps/ingest
}

// BAD — bare sentence inside tag (use > or a schema slot)
@integration {
  Identity verification provider webhooks
}
```

Use `maps_to` (underscore slot) consistently — not `maps to` (legacy alias may warn).

See [registry.md — Tags](registry.md#tags) for per-tag slot tables; package tags use [packages.md — define tag](packages.md#define-tag).

---

## Version declaration

```pactia
pactia 1.0
```

The first line declares the language version. The compiler rejects unsupported major versions.

---

## Program structure

A valid program contains exactly one `product` block. Declarations live inside `product` or nested `module` / `service` / `data` blocks.

Single-file and [workspace](language-spec.md#workspace-layout) layouts compile to the same IR.

```
Program     ::= VersionDecl ProductDecl
ProductDecl ::= "product" Identifier "{" ProductBody "}"
ProductBody ::= { ProductItem }
ProductItem   ::= ProseLine | ProductTag | ModuleDecl | UseDecl | ImportDecl | DefineDecl | YamlDecl | Comment
ProseLine     ::= ">" ProseText
Comment       ::= LineComment | BlockComment
LineComment   ::= "//" ...
BlockComment  ::= "/*" ... "*/"
ModuleDecl  ::= "module" Identifier "{" ModuleBody "}"
ServiceDecl ::= "service" Identifier "{" ServiceBody "}"
DataDecl    ::= "data" "{" DataBody "}"
DefineDecl  ::= "define" DefineBody
DefineBody  ::= "template" Identifier "(" ParamList ")" "{" TemplateBody "}"
              | "macro" Identifier MacroBody
              | "tag" Identifier TagDefBody
```

`define template` may appear at `product`, `module`, or `service` scope. **`define macro` and `define tag` are package-authoring only** — valid at file root of `index.pactia` (or package workspace entry), not inside consumer `product { }`. Template invocations use `#[templateName(Arg, ...)]` — never bare call lines.

**Not allowed:** `define Name = string`, `define Name = uuid`, or `define Name = Enum.VALUE`. Pactia has no user-defined type or constant aliases — use scalar kinds on fields (`vin: string`) and enum refs inline (`VehicleStatus.DECOMMISSIONED`).

---

## Product block

```pactia
pactia 1.0

import @pactia/protocol-rest;
import @pactia/rust-anb;

product FleetManagement {
  > Platform for tracking vehicles and managing fleets

  @stack rust-anb { }
  @topology { mode: microservices, }
  @tenancy { mode: single, }

  module fleet {
    ...
  }
}
```

First `> prose` line after `{` is the product description. Stack semver is declared in `kabol.toml` / `kabol.lock` — not inside `@stack { }`. See [registry.md](registry.md#tags) and [platform.md](platform.md#stack-versions).

---

## Roles and capabilities

```pactia
module fleet {
  @actor admins {
    role: Admin,
    capabilities: [manage_fleets, manage_users],
  }

  @actor customers {
    role: Customer,
    capabilities: [track_vehicles, view_history],
  }
}
```

`@actor` lowers to `business.actors[]` (capabilities are `snake_case` identifiers in IR). The same roles gate API `@auth { }` and UI visibility on every surface. See [Authorization](#authorization).

---

## Authorization

Authorization has **two layers** — application roles and party-scoped access on resources.

### Application roles

| Construct | Scope | IR |
| --- | --- | --- |
| `@actor <id> { role: Role, capabilities: [...] }` | `module` | `business.actors[]` |
| `@auth { roles: [Role, ...] }` | `@api` / `@api` | `authorization.roles` |
| `@public` | `@api` / `@api` | `authorization.type: PUBLIC` |

Roles declared in `@actor` must match identifiers used in `@auth { roles: [...] }`. Capabilities are documentation and AI context unless linked to `@test` / `@must`.

### Party scope (resource-bound)

Party macros express **row- or field-level** rules on top of role checks:

| Macro | Meaning | IR |
| --- | --- | --- |
| `#[owner]` | Actor may access rows they own (FK inference, e.g. `customerId`) | `ownership.scope: OWN_ROWS` |
| `#[buyer]` | `auth.sub == resource.buyerId` | `ownership.scope: PARTY_BUYER` |
| `#[seller]` | `auth.sub == resource.sellerId` | `ownership.scope: PARTY_SELLER` |
| `#[participant]` | Buyer or seller on the resource | `ownership.scope: PARTY_PARTICIPANT` |

Stack packages declare baseline JWT claim names in `platformLaw.jwtClaims` (e.g. `sub`, `role`, `iat`, `exp`). Party macros lower to conformance checks on those claims — not to middleware implementation (below [the intent line](overview.md#the-intent-line)).

```pactia
@auth { roles: [Customer, Admin] }
#[list]
#[paginated]
#[owner]
@api list_vehicles {
  method: GET,
  path: "/api/v1/vehicles",
}
```

See [registry.md — Authorization / ownership macros](registry.md#authorization--ownership-party-scope).

---

## Rules, tests, and outcomes

```pactia
service FleetService {
  @rule domain_single_customer {
    > Every vehicle belongs to exactly one customer.
  }

  @test customer_cannot_register {
    name: "Customer cannot register vehicles",
    when: "Customer is logged in and POST /api/v1/vehicles",
    then: "status is 403",
  }

  @must payment_failed_release {
    on: payment_failed,
    > inventory reservation is released
    > order status becomes cancelled
  }
}
```

| Syntax | IR |
| --- | --- |
| `@rule <id> { > ... }` | `rules[]` (enforced) |
| `> ...` prose | `prose[]` (guidance / AI context) |
| `@test <id> { name:, when:, then: }` | `scenarios[]` |
| `@must <id> { on: trigger, > outcomes... }` | `obligations[]` |

---

## Endpoints (protocol packages)

REST APIs are declared with **`@api { }`** from `import @pactia/protocol-rest` — not bare `GET` / `POST` lines.

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

  @auth { roles: [Admin] }
  #[create]
  #[idempotent]
  @body CreateVehicleRequest
  @returns CreateVehicleResponse
  @status 201
  @emit vehicle.created
  @throws { names: [ValidationFailed, Conflict] }
  @api create_vehicle {
    method: POST,
    path: "/api/v1/vehicles",
  }
}
```

The kernel routes `@api { }` to `input/services/*.yaml` using the protocol package schema. gRPC and GraphQL attach `@grpc { }` / `@graphql { }` alongside the same logical `@api { }` operation — see [platform.md](platform.md#protocol-packages).

---

## `data` block

One place for entities, enums, relations, state machines, and request/response shapes — **all as kernel tags**:

```pactia
data {
  @enum VehicleStatus {
    values: [ACTIVE, INACTIVE, DECOMMISSIONED],
  }

  @entity Vehicle {
    @pk
    id: uuid,
    @fk { entity: Customer }
    @index
    customerId: uuid,
    @unique
    vin: string,
    status: VehicleStatus,
    label: string,
  }

  @entity Customer {
    @pk
    id: uuid,
    name: string,
    @pii
    @unique
    email: string,
  }

  @relation customer_owns_vehicles {
    from: Customer,
    to: Vehicle,
    verb: owns,
    cardinality: many,
  }

  @states vehicle_lifecycle {
    entity: Vehicle.status,
    transitions: [
      { from: ACTIVE, to: INACTIVE },
      { from: INACTIVE, to: DECOMMISSIONED },
    ],
  }

  @rule domain_single_customer {
    > Every vehicle belongs to exactly one customer.
  }
}
```

Field lines (`name: Type`) appear **inside `@entity { }` bodies** — parsed by the `@entity` tag schema, not by kernel HTTP matching.

Field modifier tags use **prefix** placement on the line above: `@pk { }`, `@fk { entity: Entity }`, `@unique { }`, `@pii { }` — see [registry.md](registry.md#tags).

Types: `uuid`, `string`, `int`, `decimal`, `bool`, `datetime`, `json`. Array types: `Type[]`. Optional fields: suffix `?`.

---

## Multi-surface

```pactia
@api {
  method GET
  path /api/v1/vehicles
  @auth { Customer, Admin }
  #[list] #[paginated] #[owner]
  @returns { VehicleListResponse }

  @web {
    @screen { vehicle-list }
    @route { /fleet/vehicles }
    #[a11y(WCAG-AA)]
    @bind { FleetService GET /api/v1/vehicles }
    Customer browses their vehicles in a paginated table
  }

  @ios {
    @screen { vehicle-list }
    @nav { FleetTab }
    @bind { FleetService GET /api/v1/vehicles }
    Customer scrolls fleet on phone with pull-to-refresh
  }
}
```

Compiler emits:

- `input/services/*.yaml` — `@api { }` blocks and nested API tags
- `input/surfaces/web.yaml`, `input/surfaces/ios.yaml`, … — surface tags nested under `@api { }`

Surface packages (`@pactia/surface-react`, `@pactia/surface-swiftui`) register extra tags — not new kernel words.

---

## Tagged blocks

```pactia
@config {
  DATABASE_URL required "PostgreSQL connection string"
  GPS_DEVICE_API_KEY secret required "Inbound device API key"
  LOG_LEVEL optional default "info"
}

@errors {
  NotFound 404 RESOURCE_NOT_FOUND "Resource does not exist"
  Forbidden 403 ACCESS_DENIED
}

@policy {
  retain GpsPosition forever because "position history audit rule"
  residency EU
}

@event {
  vehicle.created payload VehicleCreatedPayload
  handler NotificationService.onVehicleCreated
  Fired when a vehicle is registered in the platform
}
```

Policy, security, observability, deployment, and guidance use the same `@tag` block pattern — see [registry.md](registry.md#cross-cutting-concerns).

Use `@guide` for AI guidance (not enforced). Use `@policy`, `@must`, `@test` for law.

---

## Imports and exports

Pactia uses a **single `import` keyword** for all dependency resolution — registry packages from kabol.io and local files alike. The compiler classifies the source by its prefix (`@scope/name` vs `./path`), not by which keyword introduced it. One mechanism, consistent syntax, no special cases — same mental model as TypeScript and Python.

There is **no separate `use` keyword**. A previous draft had `use` for registry packages and `import` for local files; that distinction added a second way to say the same thing for no semantic gain.

**`from` and `as`** are syntax tokens inside `import` statements, not kernel keywords. Outside import context they are plain English and the parser ignores them.

Tags and macros are **workspace-scoped** — kernel, stack, or explicit `import`. See [registry.md — Workspace registry](registry.md#workspace-registry).

**Versions:** semver ranges in `kabol.toml`; exact pins in `kabol.lock` — like Cargo. **`import` never carries a version.**

### Import forms

```pactia
import @pactia/kyc-compliance;
import * from @pactia/kyc-compliance;
import { sanctions_check, sanctions_screen } from @acme/fintech-rules;
import sanctions_check from @pactia/kyc-compliance;
import @pactia/kyc-compliance as kyc;
import sanctions_check as screen_check from @pactia/kyc-compliance;
import "./features/place-order.pactia";

@kyc::sanctions_check { level: enhanced, }
```

| Form | Meaning |
| --- | --- |
| `import @scope/name;` | Package prelude (default exported symbols) |
| `import * from @scope/name;` | All exported registry tags, macros, and AST symbols |
| `import { a, b } from @scope/name;` | Listed symbols only |
| `import symbol from @scope/name;` | One tag, macro, entity, enum, or service |
| `import @scope/name as alias;` | Package qualifier — `@alias::tag`, `#[alias::macro]`, `alias.Type` |
| `import symbol as alias from @scope/name;` | Rename one imported symbol |
| `import "./path.pactia";` | Merge kernel AST from a local file (relative to importing file) |

Declare dependency first: `kabol add @pactia/kyc-compliance@^1.0` → `kabol.toml` + `kabol.lock`.

| `import` in file | Registry visible in |
| --- | --- |
| `product.pactia` | Whole workspace |
| `module.pactia` | Module subtree |
| `service.pactia` | Service subtree |
| Single-file program | Entire file |

`VERSION_IN_IMPORT` if semver appears in `import`. `DEPENDENCY_NOT_DECLARED` without `kabol.toml` entry. `REGISTRY_COLLISION` when two imports expose the same unqualified name. See [packages.md](packages.md).

### Export

Package authors mark individual entities, enums, services, tags, macros, and other symbols as **exported**; everything else in the package is private and cannot be imported by consumers.

```pactia
export define tag sanctions_check {
  scope endpoint
  body { level: string, provider: string @optional { }, }
  lowers { > input/security-policy.yaml sanctions_checks[] }
}

export define macro sanctions_screen {
  expands { @sanctions_check { level: enhanced, } }
}

data {
  export @entity Verification {
    status: KycStatus,
  }
}
```

`pactia package build` validates `export` modifiers and lowers them into the publish manifest (`registry.tags[]`, `registry.macros[]`, IR export sections). Consumers import only exported symbols — `IMPORT_NOT_EXPORTED` otherwise.

Products do **not** use `export` — they consume packages. See [packages.md — Authoring packages](packages.md#authoring-packages).

### Stack vs domain packages

| | Stack package | Domain / protocol package |
| --- | --- | --- |
| Selected via | `@stack { }` on `product` | `import @scope/name` |
| Contains | Platform defaults, CI, observability | Domain patterns, wire formats, rules |

See [platform.md](platform.md#stack-packages).

---

## `define` — templates and package registry

`define` has three roles. After expansion or package build, consumer products contain only kernel constructs — no `define` nodes in IR.

| Form | Where allowed | Purpose |
| --- | --- | --- |
| `define template name(...) { }` | Product or package | Repeat kernel blocks locally — provenance `DEFINE` |
| **`define macro name { expands { } }`** | **Package source only** | Register `#[name]` — lowers to `macros[]` at `pactia package build` |
| **`define tag name { scope body lowers }`** | **Package source only** | Register `@name { }` — lowers to `tags[]` + JSON Schema at package build |

Products **invoke** package macros and tags via `#[name]` and `@name { }` after `import @scope/package`. Products must not register new global macros or tags — publish a package instead. See [packages.md](packages.md#package-registry-define-macro--define-tag).

### Template — repeated kernel blocks

```pactia
define template fleet_list(path, ListDto) {
  @api {
    method GET
    path path
    @auth { Customer, Admin }
    #[list] #[paginated] #[owner]
    @returns { ListDto }
    @errors { Forbidden }

    @web {
      @screen { vehicle-list }
      @route { /fleet/vehicles }
      #[a11y(WCAG-AA)]
      @bind { FleetService GET path }
    }
  }
}

module fleet {
  service FleetService {
    #[fleet_list(/api/v1/vehicles, VehicleListResponse)]
  }
}
```

After expansion → `@api { }` blocks with `#[macro]` invocations intact; macro expansion runs in the next phase.

**Rules:**

- Template bodies may only contain **kernel tags** and `#[macro]` references
- Parameters are identifiers — entity names, paths, DTO names — not arbitrary code
- Expansion is **pure** (same input → same output)
- Invocation is always `#[templateName(Arg, ...)]` — never a bare call line
- Errors cite template name and invocation site
- Prefer `define template` for **3+ repetitions**; single endpoints stay explicit `@api { }`

### Policy template

```pactia
define template audit_retention(Entity, reason) {
  @policy {
    retain Entity forever because reason
  }
}

#[audit_retention(GpsPosition, "position history audit rule")]
```

### `define macro` — package registry (not in consumer products)

Authors register **`#[macro]`** patterns in package `index.pactia`. `pactia package build` compiles them into `pactia.package.yaml` → `macros[]`. Consumers never write `define macro` in a product file.

Full grammar for registry header `expands`: [packages.md](packages.md#package-authoring).

```pactia
pactia 1.0
// Package: @acme/fintech-rules

define macro sanctions_screen {
  expands {
    @sanctions_check { level enhanced }
  }
}

define macro list {
  expands {
    @cursor { default 20 max 100 }
    #[paginated]
  }
}
```

| Rule | Detail |
| --- | --- |
| `expands { }` body | Only `@tag { }`, `#[macro]`, and kernel lines allowed — same purity rules as stack `expands_to` |
| Expansion targets | Must reference **kernel tags** or **tags registered in the same package** |
| Consumer syntax | `#[sanctions_screen]` after `import sanctions_screen from @acme/fintech-rules;` |
| Override precedence | Stack package > explicit `import` packages > `@pactia/*` std packages > pactiac defaults — [registry.md](registry.md#macros) |
| Hand-authored YAML | `macros[]` in manifest remains valid; build **merges** with lowered `define macro` |

| Error | Condition |
| --- | --- |
| `DEFINE_MACRO_IN_PRODUCT` | `define macro` inside consumer `product { }` |
| `MACRO_EXPANSION_INVALID` | `expands { }` references unknown tag or macro |

### `define tag` — package registry (not in consumer products)

Authors register new **`@tag { }` names**, block schemas, allowed scopes, and IR routing in package source. Build emits `tags[]` (or `extensions[]`) plus `schemas/<name>-v1.json`.

Full grammar for registry headers `scope`, `body`, `lowers`: [packages.md](packages.md#package-authoring).

```pactia
pactia 1.0
// Package: @acme/fintech-rules

define tag sanctions_check {
  category compliance
  scope endpoint

  body {
    level: string
    provider: string @optional { }
  }

  lowers {
    input/security-policy.yaml sanctions_checks[]
    input/services/{serviceKebab}.yaml endpoints[].sanctions
  }
}
```

| Block | Meaning |
| --- | --- |
| `scope` | One or more of: `product`, `module`, `service`, `endpoint`, `field` — where `@sanctions_check { }` may appear |
| `body { }` | Field declarations — lowered to JSON Schema for tag body validation |
| `lowers { }` | One line per emission target: `<ir-file> <json-path>` — paths must be from the `@pactia/schema` allowlist |

Consumer after `import { sanctions_check, sanctions_screen } from @acme/fintech-rules;`:

```pactia
@api {
  method POST
  path /api/v1/transfers
  #[sanctions_screen]
  @sanctions_check { level enhanced provider "refinitiv" }
}
```

Without `import` → **`TAG_UNKNOWN`**. With `import` → body validated against package schema → routed per `lowers { }` with provenance `PACKAGE`.

| Error | Condition |
| --- | --- |
| `DEFINE_TAG_IN_PRODUCT` | `define tag` inside consumer `product { }` |
| `TAG_LOWERS_INVALID` | `lowers { }` targets a path not in the schema allowlist |
| `TAG_SCOPE_VIOLATION` | `@tag` used outside declared `scope` |

See [packages.md](packages.md#extensibility) and [registry.md#tags](registry.md#tags).

---

## Events and handlers

Declare events and their consumers inside **`@event { }`** — not as bare `on ... ->` lines.

```pactia
@event vehicle.created {
  payload: VehicleCreatedPayload,
  handler: NotificationService.onVehicleCreated,
  > Fired when a vehicle is registered in the platform
}

@event position.received {
  payload: GpsIngestRequest,
  handler: NotificationService.onPositionReceived,
  > Fired when a GPS device pushes a valid position update
}
```

| Field in `@event` | Meaning |
| --- | --- |
| `payload:` | Payload type in `communication.yaml` |
| `handler:` | Consumer — lowers to `eventHandlers[]` on the target service |
| `> sentence` | Description for AI and generated docs |

Producers attach with `@emit { event.name }` on `@api { }` blocks (see [registry.md](registry.md#tags)).

A standalone line such as `on vehicle.created -> NotificationService.onVehicleCreated` **outside** `@event { }` is **prose only** — useful for AI context, not compiled to `eventHandlers[]`. For enforceable wiring, put the `handler` line inside `@event { }`.

---

## `yaml` embed

The `yaml` keyword embeds **raw YAML** inside a `.pactia` file. Content inside triple-quoted heredocs (`""" ... """`) is not tokenized as Pactia.

| Mode | Syntax | Purpose |
| --- | --- | --- |
| **Product merge** | `yaml merge <target> """..."""` | Extend contract IR when compiling a product |
| **Package authoring** | `yaml package/<section> """..."""` | Define publishable stack or protocol package fragments |

Allowed merge targets: `project.yaml`, `business.yaml`, `domain.yaml`, `integrations.yaml`, `communication.yaml`, `policies.yaml`, `config.yaml`, `scenarios.yaml`, `services/<kebab-name>.yaml`.

Prefer kernel syntax for product intent. Use `yaml` when the kernel has no construct (stack `platformLaw`) or for package authoring. YAML anchors and aliases are forbidden in embeds.

Embeds are **above [the contract line](overview.md#the-intent-line)** only when they describe contract facts. Implementation hints must not appear in product `yaml merge` blocks.

---

## Provenance

Every IR field is tagged with one of:

| Provenance | Meaning |
| --- | --- |
| `Pactia` | Written directly by the author |
| `INFERRED` | Derived by a documented deterministic rule |
| `STACK_DEFAULT` | Supplied by the stack package |
| `PACKAGE` | Supplied by an `import` (merged AST or package tag registry) |
| `MACRO` | Supplied by `#[macro]` expansion (see expanded tags in IR) |
| `DEFINE` | Supplied by `define template` expansion |
| `GUIDANCE` | `@guide { }` or non-enforced prose |
| `GENERATED` | Optional `bsc expand` (LLM) narrative — never overrides formal IR |
| `NOT_DERIVABLE` | IR slot exists but Pactia does not contain the fact |

`NOT_DERIVABLE` marks [the contract line](overview.md#the-intent-line) — not a defect.

---

## Compile pipeline

```
1. Parse: nine kernel keywords + three line kinds (tag, macro, prose)
2. Assemble workspace (if multi-file) — see [language-spec.md#workspace-layout](language-spec.md#workspace-layout)
3. Resolve `@stack { }` + `import`; build effectiveRegistry (kernel + package tags[] + macros[])
4. Expand define (templates only) → kernel tags
5. Expand #[macro] using effectiveRegistry (includes #[templateName(...)])
6. Validate @tag { } bodies (kernel + package JSON schemas)
7. Lower @tags → structured IR fields
8. Lower prose → scoped strings
9. Lower @api / @entity / @relation / … → services/*.yaml, domain.yaml
10. Lower @web/@ios/... → surfaces/*.yaml; resolve @bind
11. Apply yaml merge embeds (schema-validated)
12. Validate @pactia/schema
13. (optional) bsc compile-workspace → specification/*.md
```

Package authors: `pactia package build` lowers `define macro` / `define tag` → manifest before publish — [packages.md](packages.md#package-registry-define-macro--define-tag).

No LLM in product compile. See [compilation.md](compilation.md).

---

## What AI implements per surface

Pactia does not generate code. It generates the **shared spec** every agent implements against — one product, many surfaces:

| IR slice | AI task |
| --- | --- |
| `services/*.yaml` + prose | Server handlers, integration tests |
| `surfaces/web.yaml` + `@bind` | Web routes, forms, hooks |
| `surfaces/ios.yaml` + `@bind` | SwiftUI screens, navigation |
| `surfaces/android.yaml` + `@bind` | Compose screens, navigation |
| `data` + field tags | Migrations, DTOs, validation (all surfaces) |
| `@test` / `@must` | Acceptance tests per surface |

---

## Anti-patterns

| Do not | Do instead |
| --- | --- |
| `flow { 1. charge 2. emit }` | `@must on failure` + `@test` |
| Use `@list` as a tag | `#[list]` macro — see [registry.md](registry.md#macros) |
| Bare `GET /path` or `POST /path` | `@api { method: GET, path: "/path", ... }` |
| `@auth Customer` (one role) | Valid shorthand at altitude 1 — or `@auth { roles: [Customer] }` for multiple |
| Bare sentence without `>` | `> sentence` on its own line |
| `@api` / `@api` in new files | `@api` with prefix decorators |
| `define template` for a single endpoint | Write `@api { }` explicitly |

---


---

## Workspace layout

### Zoom levels

Each level answers one question. Each level can be its own file — one AI context window per task.

| Level | Answers | Typical owner | File role |
| --- | --- | --- | --- |
| **Product** | What system exists? Who uses it? What stack? | CTO / Architect | `product.pactia` |
| **Module** | What capability group? What events cross services? | Tech Lead | `module.pactia` |
| **Service** | What deployable unit? What shared config? | Senior Engineer | `service.pactia` |
| **Feature** | What API? What rules? What outcomes? | Engineer | `features/*.pactia` |
| **Entity** | What data exists? | Engineer | `entities/*.pactia` |

**Feature** files hold `@api { }` API contracts (nested tags + prose). **Entity** files hold `data { }` blocks with `@entity { }`, `@enum { }`, etc.

---

### Directory layout

```
my-product/
  pactia.workspace.yaml       # optional — root paths, module list
  product.pactia              # product + stack + global imports

  modules/
    commerce/
      module.pactia           # roles, module rules, @event { }, depends_on
      order-service/
        service.pactia        # thin: name, config, feature/entity imports
        features/
          place-order.pactia
          cancel-order.pactia
        entities/
          order.pactia
          order-item.pactia
      payment-service/
        service.pactia
        features/
          charge.pactia
        entities/
          payment.pactia

    identity/
      module.pactia
      auth-service/
        service.pactia
        features/
          login.pactia
```

All files use the **`.pactia`** extension and the same kernel grammar. Folder names are convention, not syntax.

---

### What each file owns

### `product.pactia`

Product metadata, stack selection, topology, tenancy, and **global imports**.

```pactia
pactia 1.0

import @pactia/protocol-rest;
import @pactia/rust-anb;

product EcommercePlatform {
  > Peer marketplace platform

  @stack rust-anb { }
  @topology { mode: microservices, }
  @tenancy { mode: single, }
}
```

**Does not declare:** per-service runtime, Dockerfiles, or replica counts — those come from the [stack package](platform.md#stack-packages) and optional `@deploy` overrides.

### `module.pactia`

Capability grouping: `@actor { }`, module-wide rules, errors, `@event { }` blocks, config, `depends_on` between modules.

```pactia
module commerce {
  depends_on: Identity,

  @actor customers {
    role: Customer,
    capabilities: [place_orders, view_orders],
  }

  @actor admins {
    role: Admin,
    capabilities: [manage_orders],
  }

  @event order.placed {
    payload: OrderPlacedPayload,
    handler: NotificationModule.onOrderPlaced,
  }

  @errors platform {
    PaymentFailed: {
      status: 402,
      code: PAYMENT_FAILED,
      message: "Payment could not be captured",
    },
  }
}
```

### `service.pactia`

Thin deployable unit: service description, prefix `#[database]` `#[cache]` `#[events]` macros, and **imports** of feature and entity files.

```pactia
#[database]
#[cache]
#[events]
service OrderService {
  > Order lifecycle

  import "./entities/order.pactia"
  import "./features/place-order.pactia"
  import "./features/list-orders.pactia"
}
```

### `features/*.pactia`

One file = one API contract = one AI task. Modifier tags and macros prefix `@api { }` — no `feature` keyword.

```pactia
@auth { roles: [Customer, Admin] }
#[list]
#[paginated]
#[owner]
@returns VehicleListResponse
@api list_vehicles {
  method: GET,
  path: "/api/v1/vehicles",

  @web vehicle_list {
    @screen { id: vehicle_list }
    @route { path: "/fleet/vehicles" }
    @bind { service: FleetService, method: GET, path: "/api/v1/vehicles" }
    > Customer browses their vehicles in a paginated table
  }
}

> Results scoped to authenticated owner

@test customer_views_vehicles {
  name: "Customer views vehicles",
  when: "Customer is logged in as owner and GET /api/v1/vehicles",
  then: "status is 200",
}
```

See [language-spec.md](language-spec.md) and [registry.md](registry.md#tags).

### `entities/*.pactia`

Persistent data in `data { }` blocks using kernel domain tags.

```pactia
data {
  @enum OrderStatus {
    values: [PENDING, PAID, SHIPPED, CANCELLED],
  }

  @entity Order {
    @pk
    id: uuid,
    @fk { entity: User }
    @index
    userId: uuid,
    status: OrderStatus,
    total: decimal,
    createdAt: datetime,
  }

  @states order_lifecycle {
    entity: Order.status,
    transitions: [
      { from: PENDING, to: PAID },
      { from: PENDING, to: CANCELLED },
      { from: PAID, to: SHIPPED },
    ],
  }

  > Order total must be zero or positive
}
```

---

### Workspace manifest (optional)

`pactia.workspace.yaml` at the repo root:

```yaml
workspaceVersion: 1
pactiaVersion: "1.0"
entry: product.pactia
modules:
  - modules/commerce
  - modules/identity
```

If omitted, `pactiac compile` discovers `product.pactia` and follows `import` edges.

---

### Compile merge order

```
1. Load pactia.workspace.yaml (if present) or discover product.pactia
2. Resolve `import` (depth-first, lockfile pins)
3. Merge module.pactia files per module path
4. Merge service.pactia + imported entities + imported features
5. Expand #[template(...)] and merge feature @api { } blocks into parent service
6. Continue standard compile pipeline (validate, lower to IR)
```

Single-file programs skip steps 3–4.

---

### AI context separation

| AI task | Read these files | Generate (below the line) |
| --- | --- | --- |
| Implement one endpoint | One `features/*.pactia` + related `entities/*.pactia` | Handler, unit tests |
| Data migration | One `entities/*.pactia` | SQL migration, ORM model |
| Service wiring | `service.pactia` + module `@config` | Middleware setup |
| Platform / deploy | Stack package + `product.pactia` | Dockerfile, Helm (from stack templates) |

Pactia defines **what must stay true**. AI owns **how** — checked by `bsc conform`.

---

### Below the line: implementation hints

Optional non-enforced guidance lives in **separate** files:

```
features/place-order.impl-hint.pactia
```

```pactia
// BELOW LINE — not enforced; provenance NOT_DERIVABLE
implementation_hint PlaceOrder {
  suggested_steps: [validate, reserve_inventory, calculate_total, create_order, charge, emit]
}
```

Never put numbered `flow {}` or step `on_failure` in feature files or `@api { }` blocks — use `@must` for enforceable outcomes.

---

### Examples

- **Monolithic (this repo):** [fleet-management-v2.pactia](../fixtures/kernel/fleet-management-v2.pactia)
- **Workspace layouts:** [pactia-lang/examples](https://github.com/pactia-lang/examples)

---
## See also

- [overview.md](overview.md) — introduction and positioning
- [registry.md](registry.md#tags) — all `@tag { }` definitions
- [packages.md](packages.md#extensibility) — `define` vs `#[macro]` decision table
- [registry.md](registry.md#macros) — `#[macro]` patterns
- [registry.md](registry.md#cross-cutting-concerns) — `@guide`, `@security`, `@observe`, `@deploy`
- [Workspace layout](#workspace-layout) — multi-file layout
- [Authorization](#authorization) — roles and party model
- [platform.md](platform.md#protocol-packages) — REST, gRPC, GraphQL packages
- [packages.md](packages.md) — pactia.io registry
- [fixtures/kernel/fleet-management-v2.pactia](../fixtures/kernel/fleet-management-v2.pactia) — full example
