# JSON schemas

Machine-readable contracts for Pactia tooling. Prose lives in [docs/](../docs/).

| Path | Validates |
| --- | --- |
| [manifests/](manifests/) | `pactia.toml`, `pactia.lock` (parsed logical shape; lockfile on disk uses `[[package]]` TOML tables) |

Compiler IR (`.json` under `input/`) has **no JSON Schema** in this repo — shape is defined in [compilation.md](../docs/compilation.md) and produced by pactiac lowering. Tag bodies validate from package `export def` field specs only.

`.pactia` examples: [pactiac test/fixtures](https://github.com/pactia-lang/pactiac/tree/main/test/fixtures) and [examples](https://github.com/pactia-lang/examples).
