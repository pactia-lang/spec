# Pactia grammar reference

**Compiler implementer reference — not required reading to write Pactia.**

Authors: [overview.md](overview.md) | [language-spec.md](language-spec.md)

---

## Program structure

```
Program           ::= VersionLine? ( ImportLine | FragmentExport | DefDecl | ProductDecl )*
VersionLine       ::= "pactia" Version
ProductDecl       ::= "product" Identifier "{" ProductItem* "}"
ProductItem       ::= AttachModule | ModuleDecl | TagLike | MacroLike | ProseLine
AttachModule      ::= "module" "(" Identifier ")" "{" AttachService* "}"
AttachService     ::= "service" "(" Identifier ")" "{" AttachModel? "}"
AttachModel       ::= "model" "(" Identifier ")"
ModuleDecl        ::= "module" Identifier "{" ModuleItem* "}"
ServiceDecl       ::= "service" Identifier "{" ServiceItem* "}"
ModelDecl         ::= "model" Identifier? "{" ModelItem* "}"
FragmentExport    ::= "export" ( ModuleDecl | ModelDecl | ServiceDecl | DefDecl | ModuleConstDecl )
ImportLine        ::= "import" PackagePath ";"
                  |   "import" "{" ImportSymbol ( "," ImportSymbol )* "}" "from" FilePath ";"
ImportSymbol      ::= "@" Identifier | "@@" Identifier | "#" Identifier | Identifier
PackagePath       ::= "@" Identifier ( "/" Identifier )*
FilePath          ::= Path
```

---

## Def declarations

```
DefDecl           ::= ExportOpt "def" DefSigil DefName DefParams? InClause? "{" DefBody* "}"
ModuleConstDecl   ::= "def" Identifier "=" ( Literal | ProseLine | MultilineProse )
ExportOpt         ::= "export" | ε
DefSigil          ::= "@" | "@@" | "#"
DefName           ::= Identifier
DefParams         ::= "(" ParamList ")"
InClause          ::= "in" InTarget ( "," InTarget )*
InTarget          ::= "product" | "module" | "model" | "service" | "field"
DefBody           ::= FieldSpec | ModifierSpec | ProseLine | MultilineProse | TagApplication | ModifierApplication | MacroApplication | AssignmentLine
ModifierSpec      ::= "modifier" ","
FieldSpec         ::= Identifier ( "," | ":" DefaultValue "," )
```

Package `index.pactia`: `export def` at file root only. Fragment files: `export module` / `export service` / `export model`. Consumer `product.pactia`: non-exported `def @` / `def #` inside `module { }` only.

---

## Tags, modifiers, and macros

```
TagApplication    ::= "@" Identifier TagBody?
TagBody           ::= "{" TagBodyItem* "}"
ModifierApplication ::= "@@" Identifier ModifierArg?
ModifierArg       ::= Identifier | "(" Identifier ")"
MacroApplication  ::= "#" Identifier MacroArgs?
MacroArgs         ::= "(" ArgList ")"
ProseLine         ::= ">" ProseText
MultilineProse    ::= ">>" ProseText ">>"
Interpolation     ::= "${" Identifier "}"
```

**Legacy (deprecated):** `#[" Identifier MacroArgs? "]"` — accept during transition; emit `LEGACY_MACRO_SYNTAX` warning.

---

## Tag body items

```
TagBodyItem       ::= ProseLine | MultilineProse | AssignmentLine | FieldDeclLine | NestedTag | ModifierApplication | MacroApplication
AssignmentLine    ::= Identifier ":" Value ","
```

---

## Implementer error codes

### Registry

| Code | Condition |
| --- | --- |
| `UNKNOWN_SYMBOL` | `@` / `@@` / `#` name not in effectiveRegistry |
| `DEF_IN_PRODUCT` | `export def` in consumer product |
| `DEF_PLACEMENT_REQUIRED` | `export def` missing `in` |
| `PLACEMENT_VIOLATION` | Use outside symbol's `in` targets |
| `REGISTRY_COLLISION` | Two imports expose same unqualified name |
| `DEPENDENCY_NOT_DECLARED` | Import without `pactia.toml` entry |
| `VERSION_IN_IMPORT` | Semver in import |
| `MACRO_UNKNOWN` | Unknown `#name` |
| `MACRO_ARGS_INVALID` | Wrong arity |
| `MACRO_EXPANSION_CYCLE` | Recursive macro |
| `MACRO_EXPANSION_INVALID` | Invalid macro def body or splice result |
| `LEGACY_MACRO_SYNTAX` | Deprecated `#[name]` bracket form (warning) |

### Workspace attach

| Code | Condition |
| --- | --- |
| `IMPORT_UNUSED` | Partial import symbol never referenced |
| `ATTACH_UNDEFINED` | Attach symbol not imported |
| `ATTACH_KIND_MISMATCH` | `module(x)` expects export module, etc. |
| `CONSTANT_UNDEFINED` | `${name}` references unknown module constant |

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
| `WIRE_INVALID` | `@api` wire fields fail imported protocol package schema |
| `UNKNOWN_SYMBOL` | Unresolved `@` / `#` / `@@` |

Author-facing subset: [language-spec.md — Author errors](language-spec.md#author-errors).

---

## See also

- [language-spec.md](language-spec.md)
- [registry.md](registry.md)
- [packages.md](packages.md#package-resolution)
