# Pactia Overview

**Pactia standardizes AI-native product intent** — from a short rule file for your coding agent to a full whole-product spec — and makes that intent **reusable through versioned packages**. It is the human-facing layer of the stack described in BSC vision.

> **The one rule that matters:** Pactia never requires more than prose. Every tag, every macro, every keyword beyond `pactia` + `product` is something you reach for when you want enforcement — never something you're forced to learn to get started.

---

## Three altitudes

Every Pactia file is legal at **altitude 0**. Tags are an opt-in upgrade, one fact at a time, never all-or-nothing.

### Altitude 0 — prose only

The smallest legal program. `>` prose inside `product { }` — at least one line describing **what the product is**, then optional agent rules. No tags, no `module`.

```pactia
pactia 1.0

product MyApp {
  > A mobile app for tracking personal fitness goals and sharing progress with friends.
  > Never commit secrets. Map errors to our envelope before returning.
  > List endpoints use cursor pagination.
}
```

### Altitude 1 — light tagging

Add structure where it helps. Keep a **product description** in `product { }`, then add one tag at a time.

```pactia
pactia 1.0

import @pactia/kernel;
import @pactia/protocol-rest;

product MyApp {
  > A mobile app for tracking personal fitness goals and sharing progress with friends.

  module fitness {
    service WorkoutService {
      @auth Customer
      @@output WorkoutListResponse
      @api list_workouts {
        > Customers browse their workout history, paginated.
      }
    }
  }
}
```

Altitude 1 may omit `method` / `path`; add them when you need wire-level IR.

### Altitude 2 — fully specified

Full enforcement surface — same file shape as [relay.pactia](../fixtures/kernel/relay.pactia) (1.2 canonical). Legacy 1.1 fleet examples remain in pactiac until refreshed.

```pactia
service FleetService {
  @auth { roles: [Customer, Admin] }
  #list
  #paginated
  #owner
  @@output VehicleListResponse
  @throws { names: [Forbidden] }
  @api list_vehicles {
    method: GET,
    path: "/api/v1/vehicles",
  }
}
```

Same language, same compiler, your chosen density. See [language-spec.md — Three altitudes](language-spec.md#three-altitudes).

---

## Philosophy

### One sentence

**Pactia is an intent language for the AI era: a small, fixed structure you fill with as much or as little precision as you want — from a few rules for Cursor or Claude Code to business, entities, stack, CI/CD, deployment, and best practices — shared across teams via versioned packages.**

### A new paradigm

Classic languages answer: *how does the machine compute this?*

Pactia answers: *what should stay true about this product — and what context should every AI session inherit?*

| Era | You write | Machine / AI does |
| --- | --- | --- |
| 3GL (C, Java) | Algorithms, types | Execute |
| 4GL / config (SQL, Terraform) | Desired state | Reconcile |
| **Pactia** | Intent (prose + structured facts) | Implement; optional gates verify what you chose to formalize |

Pactia does **not** replace Rust, React, or Swift. It sits above them as a durable, versioned layer between humans, AI agents, and generated code.

### AI-native — and AI model agnostic

**AI-native** means the language is built for how software is authored now: permanent `.pactia` files instead of ephemeral chat, scoped context per `module` / `service`, and [packages](packages.md) as **shareable prompts** on pactia.io.

**AI model agnostic** means Pactia is **not** tied to any specific model (GPT, Claude, Gemini, …) or coding platform (Cursor, Claude Code, Copilot, custom agents). The same compiled output must work everywhere.

That is why `pactiac` lowers Pactia to **AI-neutral JSON IR** (`input/**/*.json`) — structured facts with provenance, not vendor prompt templates. **BSC** then:

1. **Renders** that IR into each target’s file conventions (`.cursor/rules`, `CLAUDE.md`, Copilot instructions, …).
2. **Optionally expands** it with an LLM — grounded in JSON IR, not free-form — to produce richer agent context.

The LLM step does **not** re-parse `.pactia` or change enforceable facts. It elaborates **guidance**: module overviews, happy-path scenarios, stack-aligned coding notes, gaps marked `NOT_DERIVABLE`. Formal tags in JSON remain the source of truth for conformance.

```
.pactia  ──pactiac──▶  input/modules/*/     (*.module.json, *.model.json, *.service.json)
                              │
                              └── bsc render ──▶  agent briefs (target profile)
                                        │
                                        └── bsc expand (LLM, optional) ──▶  richer agent context
                                              grounded in JSON · provenance: GENERATED
```

| Phase | Deterministic? | Role |
| --- | --- | --- |
| `pactiac compile` | Yes | Lower intent to neutral IR |
| `bsc render` (templates) | Yes | Map IR → target file layout |
| `bsc expand` (LLM) | Optional / cacheable | Enrich for readability; never override `Pactia` / `MACRO` facts |

Humans author once. The pipeline adapts to the consumer — and can make a 50-line Pactia file feel like a 200-page spec in your agent’s inbox, without bloating the source.

### Graded intent — three altitudes

See [Three altitudes](#three-altitudes) above. In short:

| Always | Your choice |
| --- | --- |
| `pactia 1.0` + `product { }` | `module`, `service`, `model` |
| Altitude 0 `>` prose in `product` — what it is + agent rules | Altitude 1: same product line + light `@tag` |
| Inside blocks: tag, macro, or prose | How much structure vs narrative |

**Heavy** reference: [relay.pactia](../fixtures/kernel/relay.pactia).

### Structure without suffocation

When you formalize something, Pactia is strict about **how** (tags and macros) so tools and packages parse the same way. When you omit formal facts, **prose is first-class** — not a hack. Standardize AI-native authoring without caging engineers or models.

### Shareable prompts (packages)

Chat prompts die in history. `pactia.toml` + `pactia.lock` pin package versions; `import @scope/name;` imports paths only — same intent in every repo and session.

### What Pactia is not

| Not | Because |
| --- | --- |
| A replacement programming language | No execution, no general computation |
| Bound to one AI vendor | BSC renders neutral JSON for any consumer |
| A mandatory full spec | Skeleton required; precision is optional |
| A non-deterministic language | Parse and compile are deterministic; implementation is free |
| Ephemeral prompt engineering | Versioned, diffable, composable, lockfile-pinned |
| LLM rewriting your `.pactia` | LLM may only **expand** BSC output from JSON IR — not mutate source or IR law |

---

## The idea

Pactia is an **intent language**, not a replacement for code. Authors declare what must stay true (when they choose to tag it) and what AI should know (prose and guidance) — across APIs, roles, data, UI intent, and policy. BSC checks implementations against the **enforceable** part of that intent (static surface checks today; runtime enforcement is planned).

| Generation | Program | Machine guarantee |
| --- | --- | --- |
| 3GL (C, Java) | Algorithms + types | Type checker rejects ill-typed programs |
| 4GL / config (SQL, Terraform) | Desired state | Reconciler converges actual to desired |
| **Pactia + BSC** | **Product intent** | **Conformance flags violations of facts you chose to formalize** |

A Pactia program is not executed. It is **compiled to module-scoped JSON IR** (`*.module.json`, `*.model.json`, `*.service.json`); BSC then **renders** and may **LLM-expand** agent briefs for your chosen toolchain — richer agent context without re-authoring `.pactia`. Conformance checks only the formalized facts in IR.

**Above [the intent line](overview.md#the-intent-line), formalized facts are deterministic and enforceable. Below it, implementation is free.** See BSC vision.

## What Pactia is

- An **AI-native intent language** — human-readable; AI is the primary consumer of compiled output, not the author of source
- **Model and platform agnostic** — same `.pactia` → same JSON IR; BSC adapts to Cursor, Claude Code, Copilot, or custom agents
- A **shareable product spec** — backend, web, mobile, desktop in one file or [workspace](language-spec.md#workspace-layout)
- **Keywords** — `product`, `module`, `service`, `model`, `import`, `export`, `def`, `in`; `@` / `@@` / `#` sigils, prose — [language-spec.md](language-spec.md)
- Composable via [packages](packages.md) on pactia.io
- Extensible via **`export def`** in packages — not new language keywords
- Stack-aware via product-level `#stack_macro` (e.g. `#rust_anb`) + `import @pactia/rust-anb` + `[stack].package` in `pactia.toml` ([platform.md](platform.md#selecting-a-stack))
- [Role-based](language-spec.md#authorization) at two layers: application roles and party roles

## What Pactia is not

| Pactia is | Pactia is not |
| --- | --- |
| An intent language for whole products | A replacement for Rust, React, or Swift |
| Graded precision (prose to full spec) | A mandate to specify everything |
| AI-neutral JSON + optional conformance | A Cursor-only or Claude-only prompt format |
| Shareable prompt standard | Ephemeral ChatGPT one-offs |
| Prose + `@` / `@@` / `#` | 50-keyword classic DSL |
| `> prose` + `//` / `/* */` comments | Bare sentences or undocumented notes in source |
| Turing-incomplete by design | A general-purpose language |
| Outcomes via `@must` / `@test` | Imperative `flow {}` scripts |

Pactia deliberately describes **less than the full system** when you want it to. Logic, topology, and tuning live [below the intent line](overview.md#the-intent-line). Numbered implementation steps belong in code or optional `implementation_hint` files.

## Design principles

1. **Intent over implementation** — declare what must stay true; never imperative logic scripts.
2. **Graded precision** — prose-only to fully tagged; the author picks the level.
3. **Fixed skeleton, open content** — block keywords + `def`; symbols from imported packages.
4. **AI-native artifact** — durable `.pactia`, not disposable chat.
5. **AI model and platform agnostic** — compile to neutral JSON; BSC renders and optionally LLM-expands per consumer.
6. **Share through packages** — reuse intent via pactia.io, not copy-paste.
7. **Strict when structured** — tags lower deterministically; conformance can hold them.
8. **Free when prose** — guidance for agents and humans unless linked to `@test` / `@must`.
9. **Small kernel, rich composition** — [packages](packages.md) add verticals; extend libraries, not grammar.
10. **Provenance** — `Pactia`, `GUIDANCE`, `NOT_DERIVABLE`, … so tools know what is law vs hint.

## Position in BSC

```
┌─────────────────────────────────────────────────────────────┐
│  Pactia (.pactia) + packages       ← humans author intent       │
├─────────────────────────────────────────────────────────────┤
│  input/manifest.json, product.json, modules/*/*.json          │
│  (AI-neutral IR)              ← pactiac compile               │
├─────────────────────────────────────────────────────────────┤
│  Stack package (pactia.io)      ← platform law (read-only)   │
├─────────────────────────────────────────────────────────────┤
│  bsc render + optional LLM expand (per target)                │
├─────────────────────────────────────────────────────────────┤
│  Implementation                ← any model / any coding agent   │
└─────────────────────────────────────────────────────────────┘
```

Pactia is how humans **author**. `pactiac` produces **vendor-neutral IR**. BSC **validates**, **renders**, and may **LLM-expand** (grounded in IR) for whichever AI coding system you use. See BSC docs.

## Who writes Pactia

| Role                         | Writes                                                    |
| ---------------------------- | --------------------------------------------------------- |
| Product / domain expert      | Prose, `> rules`, `@actor { }`                          |
| Senior architect / tech lead | `model`, `service`, `@api { }`, `@tag { }`, `#macro`, `@@modifier`, `@surface { }`, `@test { }` |
| Platform team                | Stack packages on pactia.io — not Pactia                   |
| Frontend / mobile leads      | `@surface { }`, `@bind { }` in same `.pactia` file |
| Community / vendors          | Pactia packages (`import @pactia/*` on pactia.io)            |
| AI coding agent              | **Never** edits Pactia — implements from IR (`*.module.json`, `*.model.json`, `*.service.json`) |

## Language version

**Pactia 1.2** — three sigils: `@` host tags, `@@` modifier tags (next `@` or field only), `#` macros. Keywords: `product`, `module`, `service`, `model`, `import`, `export`, `def`, `in`. Multi-file workspaces: **import + attach** (`export module` fragments, `module(name) { service(…) { model(…) } }`). Monoliths inline all blocks in one file. See [language-spec.md](language-spec.md#migration-from-11).

Grammar: [language-spec.md](language-spec.md)

## Minimal example

A mid-altitude program: actors and entities are tagged; **payment lifecycle is prose** — rich context for agents without forcing every edge into `@transition` until you need conformance.

```pactia
pactia 1.0

import @pactia/kyc-compliance;
import @pactia/protocol-rest;
import @pactia/rust-anb;

product P2PExchange {
  > Peer-to-peer crypto/fiat marketplace with escrow

  #rust_anb
  @topology { mode: microservices, }

  module exchange {
    @actor traders {
      role: Trader,
      capabilities: [create_offers, take_trades],
    }

    @actor admins {
      role: Admin,
      capabilities: [resolve_disputes],
    }

    model {
      @enum TradeStatus {
        values: [PAYMENT_PENDING, PAYMENT_SENT, PAYMENT_CONFIRMED],
      }

      @entity Trade {
        @@pk
        id: uuid,
        buyerId: uuid,
        sellerId: uuid,
        status: TradeStatus,
      }

      @states trade_payment {
        entity: Trade.status,
        > PAYMENT_PENDING — buyer has accepted the trade; fiat payment not yet acknowledged.
        > PAYMENT_SENT — buyer marked payment sent; seller reviews off-platform receipt.
        > PAYMENT_CONFIRMED — seller confirmed; escrow may release crypto to the buyer.
        > Valid edges: PAYMENT_PENDING → PAYMENT_SENT (buyer only), PAYMENT_SENT → PAYMENT_CONFIRMED (seller or system after escrow check).
      }
    }

    service TradeService {
      @auth { roles: [Trader] }
      #buyer
      @@input MarkPaymentSentRequest
      @@output TradeResponse
      @api mark_payment_sent {
        method: POST,
        path: "/api/v1/trades/:id/mark-payment-sent",
        > Buyer asserts fiat was sent; moves trade PAYMENT_PENDING → PAYMENT_SENT.
        > Idempotent if already PAYMENT_SENT. Reject if buyer is not party to the trade.
      }
    }

    @event trade.payment_confirmed {
      payload: TradePaymentConfirmedPayload,
      handler: EscrowService.onPaymentConfirmed,
      > Fired when trade reaches PAYMENT_CONFIRMED and escrow can release funds.
    }
  }
}
```

When you need **enforceable** state edges, add `@transition { from, to }` on `@api` and a `transitions: [...]` array on `@states` — same IR path as prose, but conformance-checked. See [relay.pactia](../fixtures/kernel/relay.pactia) for altitude 2.

## Coherence with BSC goals

| Goal               | Pactia contribution                   | BSC contribution                                     |
| ------------------ | ------------------------------------- | ---------------------------------------------------- |
| No ambiguous APIs  | `@@input`, `@@output`, `#macro` on endpoints | Schema validation, OpenAPI render                    |
| No auth guesswork  | `@actor { }`, `@auth { }`, `#owner` / `#buyer` | `security.md`, route rules                           |
| No stack drift     | `#rust_anb` + `import` + `[stack].package` + `pactia.lock` | Stack package merge + [pactia.lock](platform.md#stack-versions) |
| Reproducible specs | `pactia.lock`, packages                | Deterministic `bsc render`                           |
| No surface drift   | `@surface { }`, `@bind { }` in same `.pactia` | Surface IR + linked API specs                      |
| AI-ready output    | Compiles to IR (services + surfaces)  | Templates + agent rules per surface                  |

## Next steps (implementation)

1. `pactiac compile` → BSC `input/` ([compilation.md](compilation.md))
2. `bsc compile-workspace` → agent briefs from IR (BSC render)
3. pactia.io package registry ([packages.md](packages.md))
4. Web UI: Pactia editor at pactia.io / local

---

## The intent line

The **intent line** (also called the contract line in BSC) divides every software product into **formalized intent** vs **free implementation** — across backend, web, mobile, and desktop. Cross-cutting tags (policy, deploy, observe) are registered in packages like any other `@tag`. Multi-file workspaces may split endpoints across `features/*.pactia` under a service — see [language-spec.md — Workspace layout](language-spec.md#workspace-layout).

### The line

```
                         ┌───────────────────────────────────────────┐
   ABOVE THE LINE        │  Contract — Pactia owns it                  │
   deterministic         │  entities · APIs · roles · UI intent · transitions    │
   enforced              │  events · policy · stack law              │
   provenance-tracked    └───────────────────────────────────────────┘
   ─────────────────────────────  conformance gate  ──────────────────────────────
   BELOW THE LINE        ┌───────────────────────────────────────────┐
   free                  │  Implementation — AI and humans own it    │
   checked, not generated│  logic · topology · tuning · copy · edges │
                         └───────────────────────────────────────────┘
```

Everything above the line is **what must stay true**. Everything below the line is **how it is made to work**. BSC compiles and enforces the top; it never generates or round-trips the bottom.

### What lives where

| Concern       | Above (contract)                                      | Below (implementation)                                |
| ------------- | ----------------------------------------------------- | ----------------------------------------------------- |
| Data          | Entities, fields, types, enums, relations, invariants | Indexes, partitioning, query plans                    |
| UI / surfaces | Screens, routes, nav, `@bind { }` links to APIs            | Component code, animations, local state              |
| API           | Operations, DTOs, params, method/path, status family  | Handler code, validation order, serialization details |
| Authorization | `@actor { }`, `@auth { }`, `#owner` / `#buyer` party macros | Middleware wiring, claim extraction code              |
| Lifecycle     | State machines, legal transitions                     | Transition side effects, retries, compensation        |
| Async         | Event names, producers, consumers, payload identity   | Delivery mechanics, batching, dead-lettering          |
| Policy        | Retention, residency, forbidden/required tech         | Backup scripts, key rotation jobs                     |
| Behavior      | (none — not expressible)                              | Business logic, algorithms, edge cases, numbered `flow` steps |
| Structure     | Service/module/feature file layout (organizational)   | Service topology, coupling, file layout                       |

If you cannot state a fact as something that must remain true regardless of implementation, it is below the line.

### Why the line exists

A specification detailed enough to fully generate a system is just code in another syntax. Model-driven engineering proved that trying to own the whole system above the line fails: the model and the code drift, round-tripping is impossible in the general case, and platform-specific detail overwhelms the abstraction.

The contract line is the fix. We deliberately keep the contract **smaller than the system**. We accept that the implementation contains decisions the contract cannot see — and instead of pretending to generate them, we **check** them.

### The honest test: NOT_DERIVABLE

The compiler (`@pactia/pactiac`) tags every lowered fact with a provenance:

| Provenance      | Meaning                                                | Side of the line |
| --------------- | ------------------------------------------------------ | ---------------- |
| `Pactia`        | Written by the author                                  | Above            |
| `INFERRED`      | Derived by a documented deterministic rule             | Above            |
| `STACK_DEFAULT` | Supplied by the stack package                        | Above            |
| `GUIDANCE`      | Author wrote `@guide` or best-practice prose           | Below (AI only)  |
| `GENERATED`     | Optional `bsc expand` (LLM) narrative from IR        | Below (AI only)  |
| `NOT_DERIVABLE` | The target IR wants it, but Pactia does not contain it | **Below**        |

`NOT_DERIVABLE` is not a defect. It is the line, made measurable. When the fleet contract compiles, the report lists exactly what an implementer (human or AI) must decide below the line — logic steps, error catalogs, indexes, summaries, payload internals. A hand-authored "golden" that fills those in is not a richer contract; it is the contract line leaking — see [rule 2 below](#rules-of-the-line).

### Rules of the line

1. **Above the line is deterministic.** No LLM decides anything above the line. Same Pactia, same contract, byte for byte.
2. **Below the line is free.** AI and humans choose logic, topology, and structure however they like.
3. **The contract never describes behavior.** No numbered `flow {}` blocks. Use `@must` for enforceable outcomes; use `@guide` or `implementation_hint` for suggested patterns (below the line).
4. **The IR is the contract, not the system.** `input/manifest.json`, `input/product.json`, and `input/modules/*/` (`*.module.json`, `*.model.json`, `*.service.json`) carry only above-the-line facts.
5. **The line is crossed only by conformance.** The sole connection between the two halves is the conformance gate: it checks the implementation against the contract and fails the build otherwise (static surface checks today; runtime enforcement planned).

### Worked example (fleet)

From [relay.pactia](../fixtures/kernel/relay.pactia):

```pactia
@auth { roles: [Customer, Admin] }
#list
#paginated
#owner
@@output VehicleListResponse
@throws { names: [Forbidden] }
@api list_vehicles {
  method: GET,
  path: "/api/v1/vehicles",
}
```

| Fact                                               | Provenance              | Side  |
| -------------------------------------------------- | ----------------------- | ----- |
| Operation exists at `GET /api/v1/vehicles`         | Pactia (`@api { }`)    | Above |
| Roles `Customer`, `Admin`                          | Pactia (`@auth { }`)    | Above |
| Ownership scope `OWN_ROWS` on `Vehicle.customerId` | MACRO (`#owner`)      | Above |
| Response shape `VehicleListResponse`               | Pactia (`@@output`) | Above |
| Cursor pagination defaults (20/100)                | STACK_DEFAULT           | Above |
| Error catalog (when `@errors { }` declared)          | Pactia                  | Above |
| The SQL query, the index it uses, the handler code | —                       | Below |
| Human narrative summaries                          | NOT_DERIVABLE           | Below |
| Error catalog (when `@errors { }` omitted on route)  | NOT_DERIVABLE           | Below |

The contract guarantees the operation, its roles, its ownership rule, and its response shape. Conformance later checks that the running endpoint matches all of those. How the endpoint is actually coded is the implementer's business.

---

## Architecture coverage

### The core tradeoff


| Approach                        | Result                                                                       |
| ------------------------------- | ---------------------------------------------------------------------------- |
| Put everything in Pactia        | Language explodes; only infra engineers can write it; defeats "super simple" |
| Put everything in stack package | Products cannot express real differences (SLOs, scale, compliance)           |
| **Split by ownership**          | Pactia stays small; output is still comprehensive                            |


**Principle: Pactia is comprehensive in output, selective in input.**

The architect writes **decisions**. The stack package provides **defaults**. The compiler **derives** the rest from `service` blocks and `model`.

```
┌────────────────────────────────────────────────────────────────┐
│  Pactia          Architect declares intent & overrides         │
├────────────────────────────────────────────────────────────────┤
│  STACK PACKAGE  Platform owns how (CI tool, tracing, Helm base)│
├────────────────────────────────────────────────────────────────┤
│  COMPILER      Derives metrics, scenarios, modules from APIs   │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              Full specification package (everything AI needs)
```

A senior architect using Pactia should finish in **one session** (~100–200 lines). The generated spec package should still contain observability, deployment, CI, testing, security — because those sections are **assembled**, not hand-typed line by line.

### Product concern matrix


| Concern                       | Architect writes in Pactia?                       | Stack package                         | Compiler derives                   |
| ----------------------------- | ------------------------------------------------- | ------------------------------------- | ---------------------------------- |
| Web / mobile / desktop UI     | **Yes** — `@surface { platform: … }`, `@bind { }` | surface std packages              | `@bind` cross-links to `@api`        |
| Product & domain              | **Yes** — core                                    | —                                     | —                                  |
| Services & APIs               | **Yes** — core                                    | pagination defaults                   | DTOs, OpenAPI, route rules         |
| Error catalog                 | **Yes** — `@errors { }`                           | fallback `[401, 403]`                 | endpoint error list from `@errors` |
| Integrations & events         | **Yes** — `@event { }` (incl. `handler`) + `@integration { }` | Kafka naming, DLQ                     | producers from `@emit`; consumers from `@event` handler |
| Auth & roles                  | **Yes** — `@actor { }` + `@auth { }` + `#owner` | JWT baseline, claims                  | route guard table                  |
| Config / secrets              | **Yes** — `@config { }`                           | 12-factor startup template            | env manifest, secrets list         |
| Security / PII / retention    | **Overrides** — `@policy { }`, field `@pii { }` `@retain { }` | OWASP, encryption, default retention  | PII list from model tags            |
| Best practices / coding style | `@guide` + stack defaults | full patterns | module / service JSON for AI |
| Observability (SLOs, alerts)  | `@observe` when non-default                   | trace sampling, metric types          | counters from `@emit` events       |
| Scalability                   | `@deploy` environment replicas                | HPA defaults, replica baselines       | per-service flags from `service`   |
| Deployment / environments     | `@deploy` when non-default                    | K8s, Helm, ports, health paths        | namespaces from env names          |
| CI/CD                         | `@deploy gate` overrides                      | provider, deploy tool, branch mapping | —                                  |
| Testing                       | **Critical scenarios** — `@test`                  | framework, coverage target            | stubs from `@test` blocks          |
| Implementation order          | **Optional** — `roadmap` (planned)                | phase templates                       | inferred from service deps         |
| Coding standards              | `@guide` or stack package                     | full patterns                         | —                                  |
| Module structure              | **No**                                            | clean architecture layers             | modules from `module` blocks       |


### CI and deployment overrides

**Yes — but as overrides, not as primary definition.**

CI/CD tool choice (GitHub Actions, ArgoCD, branch → environment mapping) belongs in the **stack package**. A fleet product on `rust-anb` should not re-specify that unless it diverges.

The architect **does** sometimes need to say:

- which environments exist and promotion rules
- gates before production (coverage, scenario pass, manual approval)
- different replica counts per environment

### Recommended Pactia syntax (1.0)

```pactia
// Stack package already says: GITHUB_ACTIONS + ARGOCD + develop→staging + main→prod

@deploy fleet {
  @environment staging {
    replicas: 2,
  },
  @environment production {
    replicas: 3,
    region: "eu-west-1",
  },

  @gate production {
    coverage: ">= 80%",
    scenarios: pass,
  },
}
```

What compiles out:

- `deployment-strategy.md` — environments, Helm, rollout
- CI section embedded from stack + gate overrides
- Links to testing and observe sections for gate criteria

**Rule:** if `@deploy` is omitted, stack package CI config is used verbatim. Zero extra Pactia lines for the common case.

### Observability — stack defaults

- Prometheus + OpenTelemetry
- Default trace sample rate, W3C propagation
- Standard metric types (counter, histogram, gauge)
- `/metrics` path and port

### Architect declares in Pactia (when needed)

```pactia
@observe fleet_slos {
  slos: [
    { service: FleetService, metric: latency_p99, target: "< 300ms" },
    { service: FleetService, metric: error_rate, target: "< 0.5%" },
    { service: FleetService, metric: availability, target: "99.9%" },
  ],
  alerts: [
    {
      id: high_error_rate,
      service: FleetService,
      when: "error_rate > 2% for 5m",
      severity: critical,
      notify: pagerduty,
    },
  ],
}
```

### Compiler derives (no Pactia required)


| From                    | Derives                                          |
| ----------------------- | ------------------------------------------------ |
| `@emit vehicle.created` | `vehicle_created_total` counter                  |
| Each `service`          | RED metrics scaffolding (rate, errors, duration) |
| Each `@integration`     | outbound call latency histogram                  |
| Stack package           | trace propagation on HTTP and Kafka              |


### Scalability

Replica counts and autoscaling overrides use `@deploy` — not a separate keyword:

```pactia
@deploy fleet {
  @environment staging {
    replicas: 2,
  },
  @environment production {
    replicas: 3,
    region: "eu-west-1",
  },
}
```

If omitted: stack package `deploymentBaseline.autoscaling` applies to all services.

`@topology { microservices }` vs `@topology { monolith }` on `product` affects whether replica blocks apply per service or per deployment unit.

### Testing — stack defaults

- Testcontainers for PostgreSQL, Redis, Kafka
- Default coverage target (e.g. 80%)
- Given/when/then format

### Architect declares (1.0)

```pactia
@test customer_cannot_read_other_vehicle {
  name: "Customer cannot read another customer vehicle",
  when: "Customer B is logged in and GET /api/v1/vehicles/:id as non-owner",
  then: "status is 403",
}
```

### Compiler derives

- One happy-path scenario per endpoint (from `@auth` + `@@output` + shapes in `model`)
- Auth negative cases for `#owner` endpoints
- Kafka emit assertions for `@emit` tags

Architect adds `@test` blocks only for **business-critical** paths; the rest are generated.

### Compiled output

Even when Pactia is minimal, the specification package must include everything an AI implementer needs — for **every surface** (API handlers, web routes, mobile screens, ops):


| Output file                            | Primary sources                                      |
| -------------------------------------- | ---------------------------------------------------- |
| `project-overview.md`                  | `product`, `@actor { }`, prose rules               |
| `model.md`                             | `model { @entity @enum @relation @states }`         |
| `api-spec.md`                          | `@api { }` + nested `@tag { }` + `#macro`       |
| `surfaces/*.json` / UI intent docs     | `@surface { }`, `@bind { }`                  |
| `module-design.md`                     | services + stack layers                              |
| `integrations.md`                      | `@integration { }` / integration prose                   |
| `security.md`                          | stack policy + `@policy { }` + `@auth { }`                   |
| `testing-strategy.md`                  | stack + derived scenarios + `@test`                  |
| `deployment-strategy.md`               | stack deploy + `@deploy` overrides                   |
| `implementation-roadmap.md`            | inferred or optional `roadmap` (planned)           |
| observe sections in deployment/testing | stack + `@observe`                                   |


The architect reads the **generated** deployment and testing docs to verify; they do not write them by hand.

### Language tiers (evolution)


| Tier         | Audience               | Pactia size | Contents                                      |
| ------------ | ---------------------- | ----------- | --------------------------------------------- |
| **Express**  | PM + tech lead         | ~50 lines   | `product`, `model`, `@api { }`, `>` rules, `@auth` |
| **Standard** | Senior architect       | ~150 lines  | + `@event`, `@integration`, `@policy`, key `@test` |
| **Extended** | Regulated / high-scale | ~250 lines  | + `@observe`, `@deploy`, `@security`, pipeline gates |


All tiers compile to the **same JSON IR depth**; Express tier relies more on inference and stack defaults.

### Decision checklist for new Pactia constructs

Before adding a keyword, ask:

1. **Does every project need to say this?** → stack package
2. **Can it be derived from services/domain?** → compiler
3. **Do only some projects differ?** → Pactia optional block
4. **Is it product/business truth?** → Pactia core (`@tag { }`)

CI tool vendor → (1). Metrics per event → (2). SLO targets → (3). Entity relations → (4) via `@relation { }`.

### Coverage summary

| Construct | Expression |
| --- | --- |
| Product, model, services | `product`, `model { @entity … }`, `@api { }` in `service` |
| State machines, party roles | `@states { }` + `#buyer` / `#owner` + `@transition { }` |
| Request/response shapes | `@@input` `@@output` |
| Packages | `import @scope/name`, local `def @` / `def #` in `module { }` |
| Integration, events, policy | `@event { }`, `@policy { }`, `@integration { }`, prose |
| Observability, deploy gates | `@observe { }`, `@deploy { }` |
| Acceptance tests | `@test { }` + When/Then |
| Multi-surface UI | `@surface { }` + `@bind { }` |

Platform overrides use registered tags from imported packages (e.g. `@pactia/kernel`, stack packages).

### Summary

- **Tradeoff:** simplicity in Pactia, comprehensiveness in compiled output.
- **CI:** `@deploy { gate ... }` overrides, not a full CI DSL; stack owns the toolchain.
- **Observability, deployment:** `@observe`, `@deploy` tags — package-defined, same as other `@tag` symbols.
- **Senior architect workflow:** write Pactia → compile → review generated spec → adjust Pactia overrides → AI implements.

See: [language-spec.md](language-spec.md), [compilation.md](compilation.md), [packages.md](packages.md)

## See also

- [language-spec.md](language-spec.md) — grammar and workspace
- [registry.md](registry.md) — symbol resolution
- [packages.md](packages.md) — composition
- [platform.md](platform.md) — stacks and protocols
- [compilation.md](compilation.md) — compiler pipeline

