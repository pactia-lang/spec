# Editor support

Status: **Specification** — VS Code / Cursor highlighting for Pactia 1.2.

Part of: [language-spec.md](language-spec.md)

---

## Install

Install the **vscode-pactia** extension from the marketplace or load from the repo `vscode-pactia/` folder.

---

## Highlighting

| Token | Examples |
| --- | --- |
| Keyword | `pactia`, `product`, `module`, `service`, `model`, `context`, `import`, `export`, `def`, `in`, `from` |
| Reserved | `view`, `interface`, `class`, `function`, `field` |
| Host tag | `@identifier` |
| Modifier tag | `@@identifier` |
| Macro invoke | `#identifier` |
| Def sigil | `def @name`, `def @@name`, `def #name` |
| Prose | `> line`, `>> block >>` |
| Interpolation | `${identifier}` |
| Comment | `//`, `/* */` |

**Legacy:** `#[identifier]` — highlight as deprecated macro form if present.

---

## Grammar rules

Highlighting follows [language-spec.md](language-spec.md) and [grammar-reference.md](grammar-reference.md):

- Block keywords open `{` … `}` nests
- `@tag { }`, `@@modifier`, and `#macro` are distinct line kinds
- Attach syntax: `module(name) { service(Name) { model(modelName) } }`, `context(symbol)`
- `context name { }` blocks: `path` plus prose; `export context`, attach `context(symbol)`
- `def` bodies: field lists, prose; macro bodies may include `@tag` / `@@tag` / `#macro` lines
- `${identifier}` in prose and macro bodies is a compile-time interpolation token

---

## Keeping grammar in sync

When the language spec changes:

1. Update [vscode-pactia/syntaxes/pactia.tmLanguage.json](https://github.com/pactia-lang/vscode-pactia/blob/main/syntaxes/pactia.tmLanguage.json)
2. Run extension grammar tests if present
3. Adjust [grammar-reference.md](grammar-reference.md) in the same change
