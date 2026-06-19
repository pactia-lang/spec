# Editor support

Status: **Specification** — VS Code / Cursor highlighting for Pactia 1.1.

Part of: [language-spec.md](language-spec.md)

---

## Install

Install the **vscode-pactia** extension from the marketplace or load from the repo `vscode-pactia/` folder.

---

## Highlighting

| Token | Examples |
| --- | --- |
| Keyword | `pactia`, `product`, `module`, `service`, `model`, `import`, `export`, `def`, `in` |
| Reserved | `view`, `interface`, `class`, `function`, `field` |
| Tag invoke | `@identifier` |
| Macro invoke | `#[identifier]` |
| Def sigil | `def @name`, `def #name` |
| Prose | `> line`, `>> block >>` |
| Comment | `//`, `/* */` |

---

## Grammar rules

Highlighting follows [language-spec.md](language-spec.md) and [grammar-reference.md](grammar-reference.md):

- Block keywords open `{` … `}` nests
- `@tag { }` and `#[macro]` / `#[macro(args)]` are distinct line kinds
- `def` bodies: field lists, prose, optional `modifier,`; macro bodies may include `@tag` / `#[macro]` lines directly
- `${identifier}` in prose and macro bodies is a compile-time interpolation token

---

## Keeping grammar in sync

When the language spec changes:

1. Update [vscode-pactia/syntaxes/pactia.tmLanguage.json](../../vscode-pactia/syntaxes/pactia.tmLanguage.json)
2. Run extension grammar tests if present
3. Adjust [grammar-reference.md](grammar-reference.md) in the same change
