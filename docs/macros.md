# Tags and macros — unified `def`

Version: **1.1**  
Status: **Specification**

Part of: [language-spec.md](language-spec.md#def--tags-and-macros)

**Tags and macros are the same mechanism.** Both are registered with `def`. The only difference is the sigil and how you invoke them.

| | Definition | Invocation |
| --- | --- | --- |
| **Tag** | `def @name … { … }` | `@name { … }` or prefix shorthand when `modifier,` in def |
| **Macro** | `def #name … { … }` | `#[name]` or `#[name(args)]` |

There is **no** `expands { }` block. The **`def` body** is the whole definition.

---

## `def` body (shared by `@` and `#`)

**In a product** — local defs inside `module { }` (no `export`):

```pactia
module RestAPI {
  def @api_v1 in service, product {
    field1,
    field2,
    field3: 100,
    nested: { sub1, sub2, },
    > guidance for authors
  }

  def @api_v2(arg1, arg2) {
    field1: arg1,
    field2: arg2,
    > ${arg1} must be ….
  }

  def @fk in field {
    field1: X,
    field2: Y,
  }
}
```

**In package `index.pactia`** — `export def` at file root only:

```pactia
export def #cursor_paginated(arg1) in service, model {
  pagination: "cursor pagination",
  max_page: arg1,
  > we use cursor pagination with the specific policy
}
```

| Rule | Detail |
| --- | --- |
| **Required fields** | bare name + comma — `field1,` — must appear at use site |
| **Optional / default** | `name: value,` — applied when omitted |
| **Nested shape** | `name: { sub1, sub2, },` |
| **Open extension** | use site may add fields **not** listed in `def`; compiler checks **required only** |
| **Prose** | `>` and `>> … >>` allowed in **both** tag and macro defs |
| **Parameters** | `def @name(a, b)` or `def #name(a)` — `${a}` in prose; macro args at `#[name(a)]` |
| **`modifier,`** | optional in `def @` only — prefix shorthand on the line before a host |

Macro body lines may also be **`@tag { }`**, **`#[nested]`**, and assignments — whatever should be **spliced in place** at `#[name]`.

---

## Placement (`in`)

```pactia
def @api_v1 in service, product { … }   // service and product blocks
def @api_v2(arg1, arg2) { … }            // omit in → all placements
def @fk in field { … }                   // field lines in model
export def #cursor_paginated(arg1) in service, model { … }
```

| `in` target | Meaning |
| --- | --- |
| `product` | inside `product { }` |
| `module` | inside `module { }` |
| `model` | inside `model { }` |
| `service` | inside `service { }` |
| `field` | on entity field lines |

**`export def` must declare `in`.** Local non-exported defs may omit `in` (all placements).

Wrong placement → **`PLACEMENT_VIOLATION`**.

---

## Tags (`def @`)

**Use:**

```pactia
@sanctions_check {
  level: enhanced,
  provider: "refinitiv",
  extra_field: true,
}
```

**Register (package):**

```pactia
export def @sanctions_check in service {
  level,
  provider,
  notes: "",
  > Screen against provider list.
}
```

---

## Macros (`def #`)

C-style **call-site substitution**. The `def #` body is spliced at `#[name]` — not a prefix on the line above.

**Register:**

```pactia
export def #sanctions_screen in service {
  @sanctions_check { level: enhanced, }
}
```

**Use:**

```pactia
service OrderService {
  #[sanctions_screen]
  @api create_transfer {
    method: POST,
    path: "/api/v1/transfers",
  }

  #[cursor_paginated(100)]
  @api list_items { … }
}
```

**Invalid** — prefix decorator:

```pactia
#[sanctions_screen]
@api create_transfer { … }
```

Parameterized macro — args bind to `def #name(arg1)` parameters; body fields may reference `arg1`.

---

## Module constants

```pactia
module example {
  def max_page = 100
  def hint = > Always validate input.

  >> Policy: ${hint} Max page is ${max_page}. >>

  service S {
    #[cursor_paginated(max_page)]
  }
}
```

Compile-time only. See [language-spec.md](language-spec.md#module-constants).

---

## Diagnostics

| Code | Condition |
| --- | --- |
| `PLACEMENT_VIOLATION` | use outside symbol's `in` targets |
| `TAG_BODY_MISSING_FIELD` | required def field missing at use site |
| `TAG_BODY_UNKNOWN_FIELD` | extra field (warning) |
| `MACRO_UNKNOWN` | unknown `#[name]` |
| `MACRO_ARGS_INVALID` | wrong arity |
| `DEF_IN_PRODUCT` | `export def` in consumer `product.pactia` |
| `DEF_PLACEMENT_REQUIRED` | `export def` missing `in` |

---

## See also

- [language-spec.md](language-spec.md)
- [registry.md](registry.md)
- [packages.md](packages.md)
