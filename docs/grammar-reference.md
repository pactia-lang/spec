# Pactia grammar reference

**Compiler implementer reference — not required reading to write Pactia.**

Authors: [overview.md](overview.md) | [language-spec.md](language-spec.md)

---

## Program structure

```
Program           ::= VersionLine? ( ImportLine | FragmentExport | ManifestExport | DefDecl | ConstantExport | ProductDecl )*
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
ManifestExport    ::= "export" Path
                   // Package index only: export "./commerce.module.pactia"
ImportLine        ::= "import" PackagePath ";"
                  |   "import" "{" ImportSymbol ( "," ImportSymbol )* "}" "from" ImportSource ";"
ImportSymbol      ::= "@" Identifier | "@@" Identifier | "#" Identifier | Identifier
ImportSource      ::= PackagePath | FilePath
PackagePath       ::= "@" Identifier ( "/" Identifier )*
FilePath          ::= Path
```

---

## Def declarations

```
DefDecl           ::= ExportOpt "def" DefSigil DefName DefParams? InClause? "{" DefBody* "}"
ModuleConstDecl   ::= "def" Identifier "=" ( Literal | ProseLine | MultilineProse )
ConstantExport    ::= ExportOpt "def" Identifier "=" ( Literal | ProseLine | MultilineProse )
  // Package index only: "export" "def" max_page "=" 100
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
TagApplication    ::= HostTagBlock | HostTagPrefix | ModifierApplication
HostTagBlock      ::= "@" Identifier HostId? "{" TagBodyItem* "}"
HostTagPrefix     ::= "@" Identifier HostPrefixArg?    /* requires modifier, in package def @ */
HostId            ::= Identifier                         /* e.g. @api list_orders { … } */
HostPrefixArg     ::= Identifier | "(" Identifier ")"   /* e.g. @auth Customer */
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
TagBodyItem       ::= ProseLine | MultilineProse | AssignmentLine | FieldDeclLine | NestedTag | MacroApplication
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
| `REGISTRY_COLLISION` | Duplicate unqualified `@` / `@@` / `#` name from any two registry sources (imports or local defs) |
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
| `IMPORT_UNUSED` | Partial import symbol never referenced (merged source) |
| `UNUSED_IMPORT` | Imported symbol never referenced in file (warning, per-file) |
| `ATTACH_UNDEFINED` | Attach symbol not imported |
| `ATTACH_KIND_MISMATCH` | `module(x)` expects export module, etc. |
| `CONTEXT_IMPORT_UNUSED` | Imported `export context` symbol never attached |
| `CONTEXT_ATTACH_UNDEFINED` | `context(x)` symbol not imported |
| `CONTEXT_ATTACH_KIND_MISMATCH` | `context(x)` expects `export context`, not module/service/model |
| `IMPORT_MISSING` | Symbol used in file but not imported (error) |
| `CONSTANT_UNDEFINED` | `${name}` references unknown module constant |
| `CONSTANT_DEF_REQUIRED` | `export name = value` missing `def` keyword |
| `EXPORT_KIND_AMBIGUITY` | Same bare name used as both constant and topology export |

### Tag bodies

| Code | Condition |
| --- | --- |
| `TAG_BODY_MISSING_FIELD` | Required def field missing |
| `TAG_BODY_UNKNOWN_FIELD` | Extra field (warning) |
| `TAG_BODY_INVALID` | Unparseable body |
| `CLAUSE_DUPLICATE_KEY` | Duplicate assignment key in any tag body |

All tags use this table only — **no tag-name-specific error codes** in pactiac.

### Package resolution

| Code | Condition |
| --- | --- |
| `PACKAGE_NOT_FOUND` | Unknown package |
| `LOCK_DIGEST_MISMATCH` | Vendored tree digest ≠ lock |
| `LOCK_STALE` | `pactia.toml` and lock out of sync |
| `LOCK_MISSING` | Dependencies declared but no `pactia.lock` |
| `LOCK_ENTRY_MISSING` | Missing lock pin |
| `DEPENDENCY_NOT_DECLARED` | Import without `pactia.toml` entry |
| `VERSION_IN_IMPORT` | Semver in import line |

### Topology packages (1.3)

| Code | Condition |
| --- | --- |
| `TOPOLOGY_DEF_FORBIDDEN` | `export def module` / `service` / `model` / `context` (use `export` without `def`) |
| `TOPOLOGY_WILDCARD_FORBIDDEN` | Bare `import @topology-pkg` (use `import { symbol } from …`) |
| `PACKAGE_EXPORT_MIXED` | Both registry and topology exports without `mixed-exports = true` opt-in |
| `EXPORT_NOT_DECLARED` | Imported symbol not found in topology package export surface |
| `TOPOLOGY_NESTED_EXPORT` | `export service` / `model` / `context` nested inside `export module { }` |
| `TOPOLOGY_MULTIPLE_ROOT_EXPORTS` | More than one root topology export per bare `.pactia` file |
| `TOPOLOGY_MANIFEST_INLINE_EXPORT` | Inline `export module` / `service` in `index.pactia` when using `export "./file"` manifest |
| `TOPOLOGY_EXPORT_FILE_MISSING` | `export "./file"` references a file that does not exist |
| `PACKAGE_PROFILE_MISMATCH` | `pactia.toml` `exports` field does not match `index.pactia` content |
| `HYBRID_PACKAGE_DISCOURAGED` | `mixed-exports = true` escape hatch; prefer separate registry + topology packages |
| `PACKAGE_IMPORT_MIXED` | Consumer imports `{ *, topologySymbol }` from a hybrid package |
| `PACKAGE_IMPORT_UNRESOLVED` | `import @pkg` in package `index.pactia` but `@pkg` not in `pactia.toml [dependencies]` |
| `PACKAGE_SYMBOL_UNRESOLVED` | `@tag`/`@@tag`/`#macro` in `export def` body doesn't resolve in the package's dependency closure |
| `PACKAGE_CIRCULAR_DEPENDENCY` | Circular dependency detected in package import graph |
| `CONSUMER_REDUNDANT_IMPORT` | Consumer explicitly imports a package already available transitively (warning) |
| `IMPORT_ALIAS_SIGIL_MISMATCH` | `@api as #endpoint` — alias sigil doesn't match original |
| `IMPORT_ALIAS_COLLISION` | Aliased name conflicts with another imported symbol |
| `IMPORT_COLLISION_RESOLVABLE` | Wildcard import creates name collision — use `as` to disambiguate (warning) |
| `TOPOLOGY_DUPLICATE_SERVICE` | Two packages export the same service name |

Author-facing subset: [language-spec.md — Author errors](language-spec.md#author-errors).

---

## See also

- [language-spec.md](language-spec.md)
- [registry.md](registry.md)
- [packages.md](packages.md#package-resolution)
