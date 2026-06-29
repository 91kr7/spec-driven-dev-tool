<!--
<instructions>
TEMPLATE: impl-notes (owned by `code-implementer`).
Path: `.sdd/impl-notes/<MOD-id>/<level>/<SPEC-ID>.impl-notes.md` (mirrors `.sdd/specs/` path exactly).
</instructions>
-->
# Impl-notes — <SPEC-ID> <Name>

> **Owned by the `code-implementer`.** Holds **concretization** omitted by SCoT (libraries, APIs, edge cases). NOT part of the gated spec; implementer must NEVER edit the spec to record these.
> **The `test-writer` never reads this file.** Goal: behavioral completeness, not byte-for-byte reproduction.

- spec: `.sdd/specs/<MOD-id>/<level>/<SPEC-ID>.spec.md`
- source: `<the file(s) from the spec's source: front-matter>`

## Chosen libraries / versions
| Concern | Choice | Version | Why |
|---------|--------|---------|-----|
| `<e.g. hashing>` | `<e.g. argon2>` | `<x.y>` | `<rationale>` |

## API bindings (SCoT call → concrete API)
| SCoT intent | Concrete call / API |
|-------------|---------------------|
| `CALL repo.save(user)` | `<orm.users.insert(...)>` |

## Language idioms / mapping decisions
- `<e.g. error_style: result → mapped to a sealed Result type / Either>`

## Edge-case handling (beyond the SCoT)
- `<edge case>` → `<how it is handled>`

## Bug-fix log (lessons learned — keep, do not delete)
| Date | Symptom | Root cause | Fix (minimal diff) | Guard so it can't recur |
|------|---------|------------|--------------------|-------------------------|
| `<ISO date>` | `<…>` | `<…>` | `<…>` | `<test / assertion added>` |

## Open concretization decisions
- `<anything provisional that a future change should revisit>`
