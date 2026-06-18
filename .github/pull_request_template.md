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
- [ ] `registry/kernel-tags.yaml`

**Primary doc / section:**

## Coherence checklist

- [ ] Examples match [fleet-management-v2.pactia](fixtures/kernel/fleet-management-v2.pactia) where applicable
- [ ] Cross-links updated (`language-spec` ↔ `registry` ↔ `packages` ↔ `compilation`)
- [ ] `CHANGELOG.md` updated under `[Unreleased]` for user-visible changes
- [ ] JSON Schema / registry catalog aligned if tags or IR shapes changed

## Downstream tooling

- [ ] No [pactiac](https://github.com/pactia-lang/pactiac) impact expected
- [ ] pactiac may need a follow-up PR (describe below)
- [ ] [vscode-pactia](https://github.com/pactia-lang/vscode-pactia) grammar / fixtures may need a follow-up PR

**Follow-up notes (if any):**

## Test plan

- [ ] Read diff for internal contradictions (same concept named two ways)
- [ ] Fixture parses as intended (manual or pactiac smoke test)
- [ ] Schema files valid JSON where changed

**Review focus:**

```text
# what reviewers should check first
```

## Breaking changes

<!-- Language, IR layout, registry categories, package kinds, etc. Write "None" if not applicable. -->

None
