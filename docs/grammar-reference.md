# Pactia grammar reference

**Compiler implementer reference — not required reading to write Pactia.**

Authors: start with [overview.md](overview.md) (three altitudes) and [language-spec.md](language-spec.md). This document holds BNF, shorthand arity rules, deprecated forms, and implementer error codes.

---

## Tag application BNF

```
ClauseTag         ::= "@" TagName TagTarget "{" ClauseBody "}"
ModifierFlag      ::= "@" TagName                              // zero args — @pk, @public
ModifierShorthand ::= "@" TagName ( Reference | Number )      // one arg — @returns VehicleDto, @status 201, @auth Customer
ModifierTag       ::= "@" TagName "{" ModifierBody "}"         // multiple args — @auth { roles: [...] }
Macro             ::= "#[" Identifier MacroArgs? "]"

TagName           ::= Identifier
TagTarget         ::= Identifier | Reference
ClauseBody        ::= ( Assignment "," | FieldDecl "," | ProseLine | NestedClauseTag )*
ModifierBody      ::= Assignment ( "," Assignment )*
```

```
DecorationLine    ::= MacroApplication | ModifierFlag | ModifierShorthand | ModifierTag
MacroApplication  ::= "#[" Identifier MacroArgs? "]"
HostLine          ::= ModuleDecl | ServiceDecl | FieldDecl | ApiDecl | WebDecl | IosDecl
```

**One form per arity** — if a tag takes one value, the body form (`{ type: X }`) does **not** parse. `MODIFIER_TYPE_WRAPPER` is removed; there is no alternate spelling.

| Arity | Canonical form | Examples |
| --- | --- | --- |
| Zero | Flag | `@pk`, `@public`, `@unique` |
| One | Shorthand | `@returns VehicleListResponse`, `@status 201`, `@emit vehicle.created`, `@auth Customer` |
| Multiple | Body | `@auth { roles: [Customer, Admin] }`, `@fk { entity: Customer }` |

---

## Unified tag body grammar

```
TagApplication    ::= "@" TagName TagTarget? "{" TagBodyItem* "}"
TagBodyItem       ::= ProseLine | AssignmentLine | FieldDeclLine | NestedTagApplication | DecorationLine
ProseLine         ::= ">" ProseText
AssignmentLine    ::= Identifier ":" AssignmentValue ","
FieldDeclLine     ::= Identifier ":" TypeRef ","
AssignmentValue   ::= String | Reference | Identifier | Number | Boolean | Array | NestedAssignBlock
```

Inside `{ }`: every structured line is a comma-terminated field/assignment **or** prose (`> ...`). Bare unquoted sentences → `TAG_BODY_INVALID` or `PROSE_PREFIX_REQUIRED`.

**Prose everywhere:** every prose line starts with `>`, including altitude 0 in `product { }`. Multiline prose = multiple `>` lines in the same block.

---

## API endpoints

**Canonical:** prefix-decorated `@api`:

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

**Deprecated (still parse, do not use in new files):**

| Form | Replacement |
| --- | --- |
| `@api name { }` | `@api name { }` |
| `@api name { auth: ..., body: ..., }` | Prefix decorators + `@api name { }` |
| `{ type: VehicleDto }` on `@returns` | `@returns VehicleDto` |

---

## Error catalog naming

| Tag | Role | Scope |
| --- | --- | --- |
| `@errors platform { NotFound: { status, code, message } }` | **Defines** catalog entries | `module` |
| `@throws { names: [NotFound, Forbidden] }` | **References** catalog on `@api` | prefix on `@api` |

---

## Implementer error codes

### Registry and workspace

| Code | Condition |
| --- | --- |
| `REGISTRY_COLLISION` | Two imports expose the same unqualified tag/macro name |
| `REGISTRY_QUALIFIER_REQUIRED` | Ambiguous name — compiler requires `@alias::name` |
| `IMPORT_NOT_EXPORTED` | `import symbol from @pkg` names a symbol not marked `export` in package source |
| `DEPENDENCY_NOT_DECLARED` | `import @scope/name` without `kabol.toml` entry |
| `VERSION_IN_IMPORT` | Semver appears in `import` statement |
| `DEFINE_TAG_IN_PRODUCT` | `define tag` inside consumer `product { }` |
| `DEFINE_MACRO_IN_PRODUCT` | `define macro` inside consumer `product { }` |
| `MACRO_UNKNOWN` | `#[name]` not registered |
| `MACRO_ARGS_INVALID` | Argument count or types fail package schema |
| `MACRO_EXPANSION_CYCLE` | Macro expansion references itself |
| `MACRO_EXPANSION_INVALID` | `expands { }` references unknown tag or macro |

### Tag and clause validation

| Code | Condition |
| --- | --- |
| `TAG_TARGET_REQUIRED` | Multi-instance tag without target |
| `TAG_SCOPE_VIOLATION` | Tag outside allowed host (scope matrix) |
| `TAG_BODY_INVALID` | Body fails tag JSON Schema |
| `TAG_BODY_UNKNOWN_FIELD` | Unknown slot in tag body |
| `CLAUSE_DUPLICATE_KEY` | Same assignment key twice in one clause |
| `CLAUSE_FIELD_SEPARATOR` | Object clause field not followed by `,` |
| `DECORATOR_MUST_PREFIX` | Modifier inside body instead of above host |
| `DECORATOR_WITHOUT_HOST` | Decoration line not followed by valid host |
| `DECORATOR_HOST_MISMATCH` | Tag `scope` does not allow the following host |
| `TAG_LOWERS_INVALID` | `lowers { }` targets a path not in the schema allowlist |

### Literals and arrays

| Code | Condition |
| --- | --- |
| `STRING_REQUIRED` | Value looks like text/path but is not wrapped in `"..."` |
| `INVALID_LITERAL` | Unquoted token breaks Identifier / Reference / Number rules |
| `PROSE_QUOTED` | Prose line incorrectly wrapped in quotes instead of `>` |
| `ARRAY_REQUIRED` | Schema expects multiple values but author used a bare scalar |
| `ARRAY_ELEMENT_INVALID` | Element inside `[...]` breaks rules |
| `SCALAR_EXPECTED` | Schema expects one value but author used `[...]` |
| `LEGACY_SLOT_SYNTAX` | Deprecated space-separated slots (`method GET`) |

### Compiler internals

| Code | Condition |
| --- | --- |
| `COMMENT_IN_IR` | Comment token appeared in lowered output (compiler bug) |
| `PROSE_PREFIX_REQUIRED` | Non-empty line looks like prose but does not start with `>` |

Author-facing subset: [language-spec.md — Author errors](language-spec.md#author-errors).
