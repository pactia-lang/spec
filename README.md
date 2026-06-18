# Pactia specification

**Pactia is an AI-native intent language** — compiled to AI-neutral YAML IR; BSC renders for any model or coding platform.

**Current version:** 1.0  
**Status:** Specification

## Hello world (altitude 0)

```pactia
pactia 1.0

product MyApp {
  > A mobile app for tracking personal fitness goals and sharing progress with friends.
  > Never commit secrets. Map errors to our envelope before returning.
  > List endpoints use cursor pagination.
}
```

No tags. No `module`. At least one `>` line should describe the product. See [overview.md](docs/overview.md#three-altitudes).

## Documents

Full index: [docs/README.md](docs/README.md) (8 documents)

| Document | Role |
| --- | --- |
| [overview.md](docs/overview.md) | Philosophy, three altitudes, intent line, architecture coverage |
| [language-spec.md](docs/language-spec.md) | Kernel grammar, workspace, authorization |
| [grammar-reference.md](docs/grammar-reference.md) | BNF and implementer error codes (not required for authors) |
| [registry.md](docs/registry.md) | Tags, macros, cross-cutting blocks |
| [packages.md](docs/packages.md) | Packages, registry authoring, extensibility |
| [platform.md](docs/platform.md) | Stacks, versions, protocols |
| [compilation.md](docs/compilation.md) | Compiler pipeline |
| [editor-support.md](docs/editor-support.md) | VS Code / Cursor syntax highlighting |

## Tooling

| Pactia spec | pactiac | pactia |
| --- | --- | --- |
| 1.0 | `>=1.0.0` | `>=1.0.0` |

## Related repositories

| Repo | Role |
| --- | --- |
| [pactiac](https://github.com/pactia-lang/pactiac) | Compiler |
| [pactia](https://github.com/pactia-lang/pactia) | Package manager (`pactia.toml`, `pactia.lock`) |
| [examples](https://github.com/pactia-lang/examples) | Sample programs |
| [vscode-pactia](https://github.com/pactia-lang/vscode-pactia) | Editor extension |

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## License

MIT — see [LICENSE](LICENSE).
