# Pactia specification

### The intent language for the AI era

**You write what must stay true. AI writes how it works.**

Pactia is an **AI-native intent language** — compiled by **pactiac** to AI-neutral JSON IR; BSC renders for any model or coding platform. Packages publish as **git repos with semver tags** (Go-style); official libraries live on GitHub ([kernel](https://github.com/pactia-lang/kernel), [pactia-io](https://github.com/pactia-lang/pactia-io)).

**Current version:** 1.2  
**Status:** Specification

---

### The intent line

```
┌─────────────────────────────────────────────┐
│  ABOVE THE LINE — Intent                    │
│  Entities · APIs · Roles · Policy · Stack   │
│  Prose · Tags · Packages · Provenance       │
└─────────────────────────────────────────────┘
────────────── conformance gate ───────────────
┌─────────────────────────────────────────────┐
│  BELOW THE LINE — Implementation            │
│  Logic · indexes · edge cases · tuning      │
│  Owned by AI and engineers. Free.           │
└─────────────────────────────────────────────┘
```

Above the line: what every session must inherit — regardless of model, engineer, or sprint.  
Below the line: how it works today. Pactia never owns it.

See [overview.md](docs/overview.md) for philosophy, three altitudes, and architecture coverage.

---

## Hello world (altitude 0)

```pactia
pactia 1.0

product MyApp {
  > A mobile app for tracking personal fitness goals and sharing progress with friends.
  > Never commit secrets. Map errors to our envelope before returning.
  > List endpoints use cursor pagination.
}
```

No tags. No `module`. At least one `>` line should describe the product. See [overview.md — three altitudes](docs/overview.md#three-altitudes).

---

## See it

Fleet management in **Pactia 1.0** — mostly prose, with tags only where structure matters (**56 lines**):

[![Pactia 1.0 fleet-management-mini example](https://raw.githubusercontent.com/pactia-lang/.github/main/profile/assets/fleet-management-example.png)](https://raw.githubusercontent.com/pactia-lang/.github/main/profile/assets/fleet-management-example.png)

Almost pure prose — same product, **`@stack` only**, no model or API tags (**37 lines**):

[![Pactia 1.0 fleet-management-prose example](https://raw.githubusercontent.com/pactia-lang/.github/main/profile/assets/fleet-management-prose-example.png)](https://raw.githubusercontent.com/pactia-lang/.github/main/profile/assets/fleet-management-prose-example.png)

[Relay fixture (1.2 canonical)](fixtures/kernel/relay.pactia)
· [Language spec](docs/language-spec.md)

Legacy fleet mini/prose/full fixtures were removed from this repo; see pactiac `test/fixtures` for 1.1 copies until refreshed.

---

## Documents

Full index: [docs/README.md](docs/README.md)

| Document | Role |
| --- | --- |
| [overview.md](docs/overview.md) | Philosophy, three altitudes, intent line |
| [language-spec.md](docs/language-spec.md) | **Language only** — grammar, `def`, blocks |
| [macros.md](docs/macros.md) | Unified `def` for tags and macros |
| [grammar-reference.md](docs/grammar-reference.md) | BNF and implementer error codes |
| [registry.md](docs/registry.md) | Symbol resolution mechanics |
| [packages.md](docs/packages.md) | Packages, `pactia.toml`, publish |
| [platform.md](docs/platform.md) | Stacks, protocols |
| [compilation.md](docs/compilation.md) | Compiler pipeline, JSON IR |
| [editor-support.md](docs/editor-support.md) | VS Code / Cursor highlighting |

## Tooling

| Pactia spec | pactiac | pactia |
| --- | --- | --- |
| 1.0 | `>=1.0.0` | `>=1.0.0` |

## The stack

```
*.pactia  ──pactiac──▶  workspace.json + slice IR  ──▶  agent context + specifications
              ▲
         pactia — vendor git deps, lockfile, build (Go-style module fetch)
```

| | Repo | Role |
| --- | --- | --- |
| Language | **spec** (this repo) | Pactia 1.2 — grammar, JSON IR, packages |
| Compiler | [pactiac](https://github.com/pactia-lang/pactiac) | Deterministic compile to module-scoped IR |
| Package manager | [pactia](https://github.com/pactia-lang/pactia) | `pactia build`, lockfiles, fetch from git |
| Kernel packages | [kernel](https://github.com/pactia-lang/kernel) | `@pactia/kernel`, `@pactia/kernel-*` |
| Stack / protocol / surface | [pactia-io](https://github.com/pactia-lang/pactia-io) | Platform and wire packages |
| Editor | [vscode-pactia](https://github.com/pactia-lang/vscode-pactia) | Syntax, tags, diagnostics |
| Examples | [examples](https://github.com/pactia-lang/examples) | Canonical workspaces |

**Model-agnostic by design.** Switch Cursor, Claude Code, or Copilot — your `.pactia` files and lockfile stay the same.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## License

MIT — see [LICENSE](LICENSE).
