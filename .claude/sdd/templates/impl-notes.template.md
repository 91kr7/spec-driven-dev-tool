# Impl-notes — <SPEC-ID> <Name>

> **Owned by the `code-implementer`.** Holds the **concretization** the
> SCoT omits — library/version, exact API usage, language idioms,
> edge-case handling, bug-fix lessons. **NOT part of the gated
> spec**; implementer must **never edit the spec** to record these.
>
> **The `test-writer` never reads this file** (test independence). Spec (behavior)
> + these notes (concretization) make the code's behavior **regenerable**
> — enough to pass the same tests and avoid re-introducing a fixed bug. Aim for
> behavioral completeness, not byte-for-byte reproduction.

- path: `.sdd/impl-notes/<MOD-id>/<level>/<SPEC-ID>.impl-notes.md` — spec's path under `.sdd/specs/` mirrored EXACTLY (`.spec.md`→`.impl-notes.md`; a module's own note → `<MOD-id>/<MOD-id>.impl-notes.md`)
- spec: `.sdd/specs/<MOD-id>/<level>/<SPEC-ID>.spec.md`
- source: `<file(s) from the spec's source: front-matter>`

## Chosen libraries / versions

| Concern        | Choice                | Version | Why                       |
|----------------|-----------------------|---------|---------------------------|
| `<e.g. hashing>` | `<e.g. argon2>`     | `<x.y>` | `<rationale>`             |

## API bindings (SCoT call → concrete API)

| SCoT intent                        | Concrete call / API                     |
|------------------------------------|-----------------------------------------|
| `CALL repo.save(user)`             | `<orm.users.insert(...)>`               |

## Language idioms / mapping decisions

- `<e.g. error_style: result → mapped to a sealed Result type / Either>`

## Edge-case handling (beyond the SCoT)

- `<edge case>` → `<how it is handled>`

## Bug-fix log (lessons learned — keep, do not delete)

| Date | Symptom | Root cause | Fix (minimal diff) | Guard so it can't recur |
|------|---------|------------|--------------------|-------------------------|
| `<ISO date>` | `<…>` | `<…>` | `<…>` | `<test / assertion added>` |

## Open concretization decisions

- `<anything provisional a future change should revisit>`
