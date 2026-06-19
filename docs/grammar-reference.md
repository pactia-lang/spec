# Pactia grammar reference

**Compiler implementer reference — not required reading to write Pactia.**

Authors: [overview.md](overview.md) | [language-spec.md](language-spec.md)

---

## Program structure

```
Program           ::= VersionLine ( ImportLine | DefDecl | ProductDecl )*
VersionLine       ::= "pactia" Version
ProductDecl       ::= "product" Identifier "{" ProductItem* "}"
ModuleDecl        ::= "module" Identifier "{" ModuleItem* "}"
ServiceDecl       ::= "service" Identifier "{" ServiceItem* "}"
ModelDecl         ::= "model" "{" ModelItem* "}"
```

---

## Def declarations

```
```
DefDecl           ::= ExportOpt "def" DefSigil DefName DefParams? InClause? "{" DefBody* "}"
ModuleConstDecl   ::= "def" Identifier "=" ( Literal | ProseLine | MultilineProse )
ExportOpt         ::= "export" | ε
DefSigil          ::= "@" | "#"
DefName           ::= Identifier
DefParams         ::= "(" ParamList ")"
InClause          ::= "in" InTarget ( "," InTarget )*
InTarget          ::= "product" | "module" | "model" | "service" | "field"
DefBody           ::= FieldSpec | ModifierSpec | ProseLine | MultilineProse | TagApplication | MacroApplication | AssignmentLine
ModifierSpec      ::= "modifier" ","
FieldSpec         ::= Identifier ( "," | ":" DefaultValue "," )
```

Package `index.pactia`: `export def` at file root only. Consumer `product.pactia`: non-exported `def @` / `def #` inside `module { }` only. Module constants use `ModuleConstDecl` (no sigil).

---

## Tags and macros

```
TagApplication    ::= "@" Identifier TagBody?
TagBody           ::= "{" TagBodyItem* "}"
ModifierPrefix    ::= "@" Identifier ( ShorthandArg | "{" … "}" )   // prefix on next host line
MacroApplication  ::= "#[" Identifier MacroArgs? "]"
MacroArgs         ::= "(" ArgList ")" | ArgList
ProseLine         ::= ">" ProseText
MultilineProse    ::= ">>" ProseText ">>"
Interpolation     ::= "${" Identifier "}"
```

---

## Tag body items

```
TagBodyItem       ::= ProseLine | MultilineProse | AssignmentLine | FieldDeclLine | NestedTag | MacroApplication
AssignmentLine    ::= Identifier ":" Value ","
```

---

## Implementer error codes

### Registry

| Code | Condition |
| --- | --- |
| `UNKNOWN_SYMBOL` | `@` / `#` name not in effectiveRegistry |
| `DEF_IN_PRODUCT` | `export def` in consumer product |
| `DEF_PLACEMENT_REQUIRED` | `export def` missing `in` |
| `PLACEMENT_VIOLATION` | Use outside symbol's `in` targets |
| `REGISTRY_COLLISION` | Two imports expose same unqualified name |
| `DEPENDENCY_NOT_DECLARED` | Import without `pactia.toml` entry |
| `VERSION_IN_IMPORT` | Semver in import |
| `MACRO_UNKNOWN` | Unknown `#[name]` |
| `MACRO_ARGS_INVALID` | Wrong arity |
| `MACRO_EXPANSION_CYCLE` | Recursive macro |
| `MACRO_EXPANSION_INVALID` | Invalid macro def body or splice result |

### Tag bodies

| Code | Condition |
| --- | --- |
| `TAG_BODY_MISSING_FIELD` | Required def field missing |
| `TAG_BODY_UNKNOWN_FIELD` | Extra field (warning) |
| `TAG_BODY_INVALID` | Unparseable body |
| `CLAUSE_DUPLICATE_KEY` | Duplicate assignment key |

### State graphs

| Code | Condition |
| --- | --- |
| `STATE_BINDING_INVALID` | Invalid `@states` binding |
| `STATE_DUPLICATE_TRANSITION` | Duplicate edge |
| `STATE_TRANSITION_UNDEFINED` | `@transition` not in graph |

### Package resolution

| Code | Condition |
| --- | --- |
| `PACKAGE_NOT_FOUND` | Unknown package |
| `PACKAGE_LOCK_MISMATCH` | Digest mismatch |
| `LOCK_ENTRY_MISSING` | Missing lock pin |
| `STACK_BINDING_MISMATCH` | Stack binding inconsistent |

Author-facing subset: [language-spec.md — Author errors](language-spec.md#author-errors).

---

## See also

- [language-spec.md](language-spec.md)
- [registry.md](registry.md)
- [packages.md](packages.md#package-resolution)
