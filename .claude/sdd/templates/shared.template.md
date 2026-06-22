<!--
  TEMPLATE — shared NON-UI abstraction spec (SHR-*, lives in specs/shared/).
  Copy this file to specs/shared/SHR-<lowerCamel>.spec.md and fill every <placeholder>.
  Authority: .claude/sdd/conventions.md (ids §2, front-matter §3, kind→FORM §3 table, status §5, ROLE block §11, change policy §8),
             .claude/sdd/scot.md (the ONLY behavioral grammar — needed ONLY when kind: service).
  Values that bind this artifact: Markdown is the source of truth (authority); reuse over repetition (DRY).
  Discover before create: read specs/indexes/* (and existing specs/shared/*) FIRST and reuse SHR-*/ENT-* by id —
    a shared abstraction exists to be reused, so promoting one over duplicating logic is the whole point.

  WHICH SECTIONS APPLY depends on `kind:` (conventions §3 table). Keep the ones for your kind, delete the rest:
    - interface → Purpose · Public interface (signatures only, NO body) · Invariants & rules · Acceptance criteria.
                  A stub/mock is AUTO-DERIVED from an interface — never spec it separately (conventions §3).
    - service   → Purpose · Public interface · Invariants & rules · SCoT body · Acceptance criteria (behavioral).
    - dto       → Purpose · Shape (field table) · Invariants & rules · Acceptance criteria.
    - enum      → Purpose · Values (value table) · Invariants & rules · Acceptance criteria.
    - type      → Purpose · Shape/definition · Invariants & rules · Acceptance criteria.
    - config    → Purpose · Keys (key/type/default table) · Invariants & rules · Acceptance criteria.

  Placeholders below are intentional. The "## Filled example" section at the bottom is a complete,
  copy-correct sample (SHR-passwordHasher, kind: interface) — keep it as a reference, delete it from a real spec.
-->
---
id: SHR-<lowerCamel>           # required — matches filename; e.g. SHR-passwordHasher (conventions §2)
name: <AbstractionName>        # required — human/PascalCase name (e.g. PasswordHasher)
kind: <interface|service|dto|enum|type|config>   # required — drives the FORM/sections (conventions §3 table)
module: MOD-<kebab>            # required — the home module this shared abstraction belongs to
status: draft                 # draft|reviewed|approved — MIRRORS the index row (index is canonical, conventions §5)
depends_on: [<SHR-*|ENT-* ids in topological order, deps first; [] if none>]
requirements: [REQ-<nnn>]     # requirement id(s) this spec serves (back-link)
source: [<path proposed from .sdd/target.md conventions, e.g. src/shared/...>]   # authoritative spec→source map
owns_sections: []             # only for co-owned aggregator files (barrel/shared types); else leave []
error_style: <result|raise>   # ONLY for behavioral kinds (service) — must equal the SCoT `error-style:` below; omit otherwise
---

# Purpose

<!-- One short paragraph: WHAT this shared abstraction is and WHY it is shared (which callers reuse it).
     Describe responsibility/contract, never implementation, library, or framework.
     For kind: interface, name the implementing class(es) that carry the actual SCoT bodies. -->
<Single-responsibility statement — the contract this abstraction defines and the duplication it prevents.>

# Public interface

<!-- The callable surface. Inputs/outputs use neutral SCoT type names (scot.md §2: String, Bool, Int, UUID, Result<T,E>, ...).
     The errors column lists error cases REGARDLESS of error_style (scot.md §6).
     For kind: interface — SIGNATURES ONLY, no behavior; the SCoT lives in the implementing class spec (conventions §3).
     For kind: dto/enum/type/config — replace this table with the corresponding Shape/Values/Keys table below. -->

| Method                        | Inputs                     | Output            | Errors                                   |
|-------------------------------|----------------------------|-------------------|------------------------------------------|
| `<methodName>(<p>: <Type>)`   | `<p>` — <meaning>          | `<ReturnType>`    | `<ErrorCase>` — <when it occurs>         |
| `<otherMethod>(...)`          | <named inputs and meaning> | `<Type>`          | `<ErrorCase>`, `<ErrorCase2>`            |

<!-- === STRUCTURAL ALTERNATIVES (use the one matching `kind:`, delete the others) ===

  ## Shape   (kind: dto / type)
  | Field        | Type            | Required | Meaning                          |
  |--------------|-----------------|----------|----------------------------------|
  | `<field>`    | `<Type>`        | yes/no   | <what it holds; constraints>     |

  ## Values  (kind: enum)
  | Value        | Meaning                                  |
  |--------------|------------------------------------------|
  | `<LABEL>`    | <what this case represents>              |

  ## Keys    (kind: config)
  | Key                  | Type      | Default        | Meaning / constraint              |
  |----------------------|-----------|----------------|-----------------------------------|
  | `<config.key>`       | `<Type>`  | `<default>`    | <what it tunes; valid range>      |
-->

# Invariants & rules

<!-- Properties that always hold for any implementation/instance, plus rules callers can rely on.
     Reference dependencies by id (e.g. ENT-user, SHR-clock). For an interface these are CONTRACT obligations
     every implementer must honour (the test-writer asserts them against any implementation). -->
- <Invariant 1 — a condition that always holds (e.g. determinism, idempotency, output format).>
- <Rule 1 — a contract obligation every implementation must satisfy, traceable to a requirement.>
- <Rule 2 — preconditions on inputs; what is rejected and how.>

# SCoT

<!-- BEHAVIORAL KINDS ONLY (kind: service). DELETE this whole section for interface/dto/enum/type/config.
     Grammar = .claude/sdd/scot.md ONLY. State error-style first (must match front-matter). One FUNCTION per public method.
     Every branching construct carries a stable [Bn] id and ALL arms must be enumerated (scot.md §4, §7).
     No library/framework names, no language syntax, no SQL — those go to .sdd/impl-notes/<id>.md. -->

```
error-style: <result|raise>

FUNCTION <methodName>(<param>: <Type>) -> <ReturnType>
  INPUT:  <param> — <meaning / shape>
  OUTPUT: <what the return value means on success and on failure>

  [B1] IF <condition> THEN
    <statement>                                  # arm B1.then
  ELSE
    <statement>                                  # arm B1.else
  END

  RETURN <value or Ok(...)/Err(...) per error-style>
END
```

# Acceptance criteria

<!-- Each ACn in Given/When/Then. For an interface, phrase ACs against the CONTRACT so any conforming
     implementation passes. For a service, cover the happy path AND every SCoT branch arm (scot.md §7). -->

- **AC1** — Given <precondition/state>, When <action/call>, Then <observable outcome>.
- **AC2** — Given <precondition/state>, When <action that triggers an error>, Then <expected error/result>.

<!-- ===================================================================== -->
<!-- The blank template ends here. Below is a complete, copy-correct       -->
<!-- example (kind: interface). Delete it when authoring a real spec.      -->
<!-- ===================================================================== -->

## Filled example

```yaml
---
id: SHR-passwordHasher
name: PasswordHasher
kind: interface
module: MOD-auth
status: draft
depends_on: []
requirements: [REQ-002]
source: [src/auth/PasswordHasher.ts]
owns_sections: []
---
```

### Purpose

`PasswordHasher` is the shared contract for one-way password hashing and
verification used across authentication flows (registration, login, password
change). It exists so that no caller re-implements hashing — every feature that
stores or checks a password depends on this single interface, which keeps the
algorithm and its parameters in one place. As an `interface` it carries
**signatures only**: the behavioral SCoT lives in the implementing class
(`CLS-argon2Hasher`), and a no-op stub returning defaults is auto-derived from
this interface for tests and not-yet-ready dependencies.

### Public interface

| Method                                 | Inputs                                                              | Output            | Errors                                                                   |
|----------------------------------------|---------------------------------------------------------------------|-------------------|--------------------------------------------------------------------------|
| `hash(plain: String) -> String`        | `plain` — the clear-text password to hash (non-empty)               | `String`          | `EmptyInput` — `plain` is empty/whitespace; `HashingFailed` — the underlying primitive could not produce a hash |
| `verify(plain: String, hash: String) -> Bool` | `plain` — clear-text candidate; `hash` — a stored hash produced by `hash` | `Bool`            | `EmptyInput` — `plain` is empty; `MalformedHash` — `hash` is not a value this interface produced |

### Invariants & rules

- `hash` is **one-way**: the returned `String` never contains the clear text and
  cannot be reversed to recover `plain`.
- For any non-empty `plain`, `verify(plain, hash(plain))` returns `true`; for any
  `other != plain`, `verify(other, hash(plain))` returns `false`.
- `hash` is **non-deterministic** across calls (each hash embeds fresh salt), so
  two calls with the same `plain` yield different `String`s — yet both verify
  `true` against `plain`.
- Empty or whitespace-only `plain` is rejected with `EmptyInput`; a `hash`
  argument not produced by this interface is rejected with `MalformedHash`.
- The interface fixes the contract only; the concrete algorithm and its cost
  parameters are an implementation concern recorded in
  `.sdd/impl-notes/SHR-passwordHasher.md` by the implementer, not in this spec.

### Acceptance criteria

- **AC1** — Given a non-empty password `plain`, When `verify(plain, hash(plain))`
  is called, Then it returns `true`, and the value returned by `hash(plain)` does
  not contain `plain` in clear text.
- **AC2** — Given a non-empty password `plain` and a different string `other`,
  When `verify(other, hash(plain))` is called, Then it returns `false`.
