<!--
  TEMPLATE — shared NON-UI abstraction (SHR-*, specs/shared/). Copy to specs/shared/SHR-<lowerCamel>.spec.md.
  Authority: conventions §2/§3/§5 + scot.md (ONLY when kind: service). Discover before create: reuse SHR-*/ENT- by id.
  Markdown is the source of truth; reuse over repetition (DRY). Delete the "## Filled example".

  Sections by kind (keep yours, delete the rest):
    interface → Purpose · Public interface (SIGNATURES only, no body) · Invariants & rules · Acceptance criteria  (stub auto-derived — never specced)
    service   → Purpose · Public interface · Invariants & rules · SCoT · Acceptance criteria
    dto/type  → Purpose · Shape (field table) · Invariants & rules · Acceptance criteria
    enum      → Purpose · Values (value table) · Invariants & rules · Acceptance criteria
    config    → Purpose · Keys (key/type/default table) · Invariants & rules · Acceptance criteria
-->
---
id: SHR-<lowerCamel>          # required — matches filename
name: <AbstractionName>      # required
kind: <interface|service|dto|enum|type|config>   # drives the FORM
module: MOD-<kebab>          # required
status: draft                # MIRRORS the index
depends_on: [<SHR-*|ENT-* ids, deps first>]
requirements: [REQ-<nnn>]    # back-link
source: [<src/shared/… from target.md>]
owns_sections: []            # only for co-owned barrel/shared-types files
error_style: <result|raise>  # ONLY for kind: service (scot.md §6) — omit otherwise
---

# Purpose
<WHAT this abstraction is and WHY it is shared (who reuses it). For kind: interface, name the implementing class(es).>

# Public interface
<Callable surface, neutral types; errors regardless of error_style. For interface = SIGNATURES ONLY (SCoT lives in the implementing class). For dto/enum/type/config, replace with the matching table below.>

| Method | Inputs | Output | Errors |
|--------|--------|--------|--------|
| `<method>(<p>: <Type>)` | `<p>` — <meaning> | `<ReturnType>` | `<ErrorCase>` — <when> |

<!-- Structural alternatives (use the one matching `kind:`):
  Shape (dto/type): | Field | Type | Required | Meaning |
  Values (enum):    | Value | Meaning |
  Keys (config):    | Key | Type | Default | Meaning / constraint | -->

# Invariants & rules
<Properties that always hold for any implementation + caller-relied rules. For an interface these are contract obligations every implementer must honour.>
- <Invariant / contract obligation, traceable to a requirement>

# SCoT
<kind: service ONLY — delete for interface/dto/enum/type/config. Grammar = scot.md; one FUNCTION per method; stable [Bn] + all arms.>

```
FUNCTION <method>(<param>: <Type>) -> <ReturnType>
  [B1] IF <condition> THEN <statement>   # B1.then
  ELSE <statement> END                   # B1.else
  RETURN <value or Ok(...)/Err(...)>
END
```

# Acceptance criteria
<Each `ACn` Given/When/Then. For an interface, phrase against the contract so any conforming implementation passes.>
- **AC1** — Given <state>, When <call>, Then <observable outcome>.
- **AC2** — Given <state>, When <error trigger>, Then <expected error/result>.

---

## Filled example — `SHR-passwordHasher` (kind: interface)

```yaml
id: SHR-passwordHasher · name: PasswordHasher · kind: interface · module: MOD-auth
depends_on: [] · requirements: [REQ-002] · source: [src/auth/PasswordHasher.ts]
```

**Purpose** — The shared contract for one-way password hashing/verification across auth flows, so no caller re-implements hashing. As an `interface` it carries signatures only; the SCoT lives in the implementing class (`CLS-argon2Hasher`); a no-op stub is auto-derived for tests.

**Public interface**

| Method | Inputs | Output | Errors |
|--------|--------|--------|--------|
| `hash(plain: String)` | non-empty clear-text | `String` | `EmptyInput`, `HashingFailed` |
| `verify(plain: String, hash: String)` | candidate + stored hash | `Bool` | `EmptyInput`, `MalformedHash` |

**Invariants** — `hash` is one-way (never contains the clear text); `verify(plain, hash(plain))` is `true`, `verify(other, hash(plain))` is `false`; `hash` is non-deterministic (fresh salt) yet still verifies; empty `plain` → `EmptyInput`; a foreign hash → `MalformedHash`. The concrete algorithm/cost is an impl-notes concern, not this spec.

- **AC1** — Given a non-empty `plain`, When `verify(plain, hash(plain))`, Then `true`, and `hash(plain)` does not contain `plain`.
- **AC2** — Given `plain` and a different `other`, When `verify(other, hash(plain))`, Then `false`.
