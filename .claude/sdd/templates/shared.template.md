<!--
<instructions>
TEMPLATE: shared NON-UI abstraction (SHR-*). Copy to `.sdd/specs/<MOD>/shared/SHR-<lowerCamel>.spec.md`.
Authority: conventions §2/§3/§5 + scot.md (ONLY when kind: service). Discover before create: reuse SHR-*/ENT- by id.
DRY. Delete the `<example>` block before saving.

Sections by kind (keep yours, delete the rest):
  interface → Purpose · Public interface (SIGNATURES only) · Invariants & rules · Acceptance criteria
  service   → Purpose · Public interface · Invariants & rules · SCoT · Acceptance criteria
  dto/type  → Purpose · Shape (field table) · Invariants & rules · Acceptance criteria
  enum      → Purpose · Values (value table) · Invariants & rules · Acceptance criteria
  config    → Purpose · Keys (key/type/default table) · Invariants & rules · Acceptance criteria
</instructions>
-->
---
id: SHR-<lowerCamel>          # required
name: <AbstractionName>       # required
kind: <interface|service|dto|enum|type|config>   # drives the FORM
module: MOD-<kebab>           # required
depends_on: [<SHR-*|ENT-* ids, deps first>]
requirements: [REQ-<nnn>]     # back-link
source: [<src/shared/… from target.md>]
owns_sections: []             # only for co-owned barrel/shared-types files
error_style: <result|raise>   # ONLY for kind: service
---

# Purpose
<instruction>WHAT this abstraction is and WHY it is shared. For interface, name implementing class(es).</instruction>

# Public interface
<instruction>Callable surface, neutral types. For interface = SIGNATURES ONLY. For dto/enum/type/config, replace with matching table below.</instruction>
| Method | Inputs | Output | Errors |
|--------|--------|--------|--------|
| `<method>(<p>: <Type>)` | `<p>` — <meaning> | `<ReturnType>` | `<ErrorCase>` — <when> |

<!-- Structural alternatives (use the one matching `kind:`):
  Shape (dto/type): | Field | Type | Required | Meaning |
  Values (enum):    | Value | Meaning |
  Keys (config):    | Key | Type | Default | Meaning / constraint | -->

# Invariants & rules
<instruction>Properties that always hold. For interface, these are contract obligations.</instruction>
- <rule>

# SCoT
<instruction>kind: service ONLY. Grammar = scot.md; one FUNCTION per method; stable [Bn] + all arms.</instruction>
```
FUNCTION <method>(<param>: <Type>) -> <ReturnType>
  [B1] IF <condition> THEN <statement>   # B1.then
  ELSE <statement> END                   # B1.else
  RETURN <value or Ok(...)/Err(...)>
END
```

# Acceptance criteria
<instruction>Given/When/Then. For interface, phrase against contract.</instruction>
- **AC1** — Given <state>, When <call>, Then <observable outcome>.
- **AC2** — Given <state>, When <error trigger>, Then <expected error/result>.

---
<example name="SHR-passwordHasher (interface)">
```yaml
id: SHR-passwordHasher
name: PasswordHasher
kind: interface
module: MOD-auth
depends_on: []
requirements: [REQ-002]
source: [src/auth/PasswordHasher.ts]
```

**Purpose**
Shared contract for one-way password hashing/verification across auth flows. No SCoT (lives in implementing `CLS-argon2Hasher`).

**Public interface**
| Method | Inputs | Output | Errors |
|--------|--------|--------|--------|
| `hash(plain: String)` | non-empty clear-text | `String` | `EmptyInput`, `HashingFailed` |
| `verify(plain: String, hash: String)` | candidate + stored hash | `Bool` | `EmptyInput`, `MalformedHash` |

**Invariants**
- `hash` is one-way
- `verify(plain, hash(plain))` is `true`
- `verify(other, hash(plain))` is `false`
- `hash` is non-deterministic yet still verifies

**Acceptance criteria**
- **AC1** — Given a non-empty `plain`, When `verify(plain, hash(plain))`, Then `true`, and `hash(plain)` does not contain `plain`.
- **AC2** — Given `plain` and a different `other`, When `verify(other, hash(plain))`, Then `false`.
</example>
