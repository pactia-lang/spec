# Pactia editor support

Syntax highlighting for `.pactia` in Cursor / VS Code.

Extension: [vscode-pactia](../../vscode-pactia/) | [language-spec.md](language-spec.md) | [registry.md](registry.md)

## Install (required)

From repo root:

```bash
./vscode-pactia/scripts/install-extension.sh
```

Packages a VSIX and installs into Cursor and VS Code (when available). Removes stale `pactia-lang.pactia-*` versions and legacy symlinks.

Then **Developer: Reload Window** and open a `.pactia` file. Bottom-right language mode must show **Pactia** (not Plain Text).

Verify grammar balance:

```bash
cd vscode-pactia && npm install && npm run test:grammar
```

## Highlighting

| Highlight as | Examples |
| --- | --- |
| **Kernel keywords (9)** | `pactia`, `product`, `module`, `service`, `data`, `import`, `export`, `define`, `yaml` |
| **Registry headers** | `scope endpoint`, `body { }`, `lowers { }`, `expands { }`, `category compliance` |
| **Clause tags (teal)** | `@entity Vehicle { }`, `@api list { }`, `@actor customers { }` |
| **Macros (purple, bold)** | `#[list]`, `#[database]`, `#[alias::macro]` — above `service` / `@api` |
| **Modifier flags** | `@pk`, `@public`, `@pii`, `@optional` — no `{ }` when empty |
| **Modifier shorthand** | `@returns VehicleDto`, `@status 201`, `@emit vehicle.created` |
| **Imports** | `import @scope/name;`, `import { a, b } from @pkg;`, `import @pkg as alias;` |
| **Qualified tags** | `@alias::sanctions_check { }` |
| **Prose (purple, italic)** | `> sentence` |
| **Strings (green)** | `"PostgreSQL connection string"`, `"/api/v1/orders"` |
| **Braces (gold)** | `{` `}` on blocks and inline objects |

## Grammar rules (v1.0.0)

1. **Multiline blocks** open only when `{` is the **last character on the line** (`@entity Vehicle {`, `product X {`).
2. **Single-line modifiers** — `@auth { roles: [...] }`, `@screen { id: x }` matched before multiline rules.
3. **Indent-aware close** — `}` at the same indent as the opening line closes the block.
4. **Inline objects** — `{ service: FleetService, metric: error_rate }` inside arrays.
5. **Package authoring** — `define tag` / `define macro` with nested `body { }`, `lowers { }`, `expands { }` registry blocks.
6. **Qualified symbols** — `@alias::tag`, `#[alias::macro]` after import @pkg as alias;`.

## Keeping grammar in sync

1. Update [language-spec.md](language-spec.md) and [registry.md](registry.md)
2. Edit [vscode-pactia/syntaxes/pactia.tmLanguage.json](../../vscode-pactia/syntaxes/pactia.tmLanguage.json)
3. Run `cd vscode-pactia && npm run test:grammar`
4. Re-run `scripts/install-extension.sh` and reload window
