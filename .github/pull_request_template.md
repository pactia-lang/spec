## Summary

<!-- What changed and why? Link related issues: Fixes #123 -->

## Type of change

- [ ] Clarification (no normative behavior change)
- [ ] Normative spec change (language, registry, IR, schemas)
- [ ] Fixture or schema artifact only
- [ ] Editorial / coherence (cross-doc links, typos, examples)
- [ ] RFC / breaking change (needs explicit review)

## Documents touched

- [ ] `docs/overview.md`
- [ ] `docs/language-spec.md`
- [ ] `docs/registry.md`
- [ ] `docs/packages.md`
- [ ] `docs/platform.md`
- [ ] `docs/compilation.md`
- [ ] `docs/editor-support.md`
- [ ] `docs/grammar-reference.md`
- [ ] `CHANGELOG.md`
- [ ] `fixtures/` and/or `schemas/`

**Tag/macro names:** not in this repo — they belong to packages (e.g. `@pactia/kernel` on pactia.io).

**Primary doc / section:**

## Coherence checklist

- [ ] Examples use 1.2 syntax: `#macro`, `@@modifier`, import + attach (not `#[…]` or `modules/*` scan)
- [ ] [relay.pactia](fixtures/kernel/relay.pactia) remains the canonical **monolith** 1.2 fixture in this repo
- [ ] Workspace attach examples point at [pactiac relay workspace](https://github.com/pactia-lang/pactiac/tree/main/test/fixtures/workspace/relay) when multi-file composition is shown
- [ ] Cross-links updated (`language-spec` ↔ `registry` ↔ `packages` ↔ `compilation`)
- [ ] `CHANGELOG.md` updated under `[Unreleased]` for user-visible changes
- [ ] [Compiler alignment (1.2)](docs/language-spec.md#compiler-alignment-12) table updated if pactiac status changed
- [ ] Field specs / IR JSON schemas aligned if shapes changed

## Downstream tooling

Pair normative language changes with compiler/editor PRs when behavior changes:

| Repo | When to open a follow-up | Typical branch |
| --- | --- | --- |
| [pactiac](https://github.com/pactia-lang/pactiac) | Parse, registry, attach, macro, or IR behavior changed | `feat/pactiac-1.2-compiler` (or successor) |
| [vscode-pactia](https://github.com/pactia-lang/vscode-pactia) | Grammar, highlighting, or snippets for new sigils (`#`, `@@`, attach) | TBD |

- [ ] No pactiac / vscode impact expected
- [ ] Companion [pactiac PR](https://github.com/pactia-lang/pactiac/pulls) linked below
- [ ] Companion vscode-pactia PR linked below (grammar / editor-support)

**Companion PRs (if any):**

- pactiac: <!-- e.g. https://github.com/pactia-lang/pactiac/pull/N — feat/pactiac-1.2-compiler -->
- vscode-pactia: <!-- if applicable -->

**Known compiler gaps (update [language-spec.md — Compiler alignment](docs/language-spec.md#compiler-alignment-12) when closed):**

- `${constant}` compile-time substitution in prose (utility exists; not wired in pactiac expand/bind)

## Test plan

- [ ] Read diff for internal contradictions (same concept named two ways)
- [ ] [relay.pactia](fixtures/kernel/relay.pactia) still matches cited examples
- [ ] Fixture parses with pactiac smoke test when syntax changed
- [ ] Schema files valid JSON where changed

**Review focus:**

```text
# what reviewers should check first
```

## Breaking changes

<!-- Language, IR layout, registry categories, package kinds, etc. Write "None" if not applicable. -->

None
