<!--
<instructions>
TEMPLATE: MODULE spec (kind: module, structural overview). Schema per conventions §3.
Partitions the system and names the entries it contains — no SCoT, no UI schematic.
Copy to `.sdd/specs/<MOD-id>/<MOD-id>.spec.md`. Register in global `modules.index.md`.
DRY. Delete `<example>` blocks before saving.
</instructions>
-->
---
id: MOD-<kebab>                 # required
name: <HumanModuleName>         # required
kind: module                    # required — drives the structural-overview FORM
module: MOD-<kebab>             # required — self-reference
depends_on: [MOD-<other>]       # topological; [] if none
requirements: [REQ-<nnn>]       # [] if purely infra
source: []                      # [] UNLESS MOD-build / MOD-schema owning infra files (§2)
owns_sections: []
---

# Purpose
<instruction>Cohesive responsibility this module owns and what falls outside it. Boundary-defining.</instruction>

# Contained entries
<instruction>Every id with `module: MOD-<this>` and matching index row. One line each. Omit empty groups.</instruction>
| Level | Entry id | What it represents (one line) |
|-------|----------|-------------------------------|
| Feature | `FEAT-<nnn>` | <use-case orchestrated here> |
| Entity | `ENT-<kebab>` | <domain entity owned here> |
| Class | `CLS-<lowerCamel>` | <service/controller/screen> |
| Component | `COMP-<lowerCamel>` | <shared UI component> |
| Shared | `SHR-<lowerCamel>` | <shared non-UI abstraction> |

# Boundaries & dependencies
<instruction>Must equal `depends_on:` and union of contained entries' cross-module deps. Acyclic (§12).</instruction>

**Depends on** — | `MOD-<other>` | <what it uses from it> |
**Exposes to** — | `MOD-<other>` | <entry ids other modules may reference> | (anything unlisted is internal)

# Conventions specific to the module
<instruction>Only rules special to THIS module, not already in conventions/target. Else: "None beyond conventions.md."</instruction>
- <module-local rule, or "None beyond conventions.md.">

---
<example name="MOD-build (Infra Exception)">
```markdown
---
id: MOD-build
name: Build & Infrastructure
kind: module
module: MOD-build
depends_on: []
requirements: []
source: [package.json, tsconfig.json, .github/workflows/ci.yml]
owns_sections: []
---

# Purpose
Mandatory scaffolding: build files, manifests, runtime config, CI, app entry, e2e harness config. Pure scaffolding — no domain dependency — builds first.

# Contained entries
| Level | Entry id | What it represents (one line) |
|-------|----------|-------------------------------|
| — | — | none — MOD-build owns its infra files directly via `source:` (§2 exception) |

# Boundaries & dependencies
**Depends on** — | — | nothing (pure scaffolding) |
**Exposes to** — | `MOD-api`/`MOD-web` | built artifacts (+ GUI: the e2e harness) |

# Conventions specific to the module
- **Config keys declared once** (key/type/default)
- **CI order fixed:** install → lint → build → test → package.
```
</example>

<example name="MOD-schema (Infra Exception)">
```markdown
---
id: MOD-schema
name: Database Schema
kind: module
module: MOD-schema
depends_on: [MOD-model, ENT-user]
requirements: [REQ-001]
source: [db/schema/V1__create_user.sql]
owns_sections: []
---

# Purpose
DB schema: forward-only schema-change scripts **derived from the entity specs in MOD-model** (never hand-authored). Ordered after the entities it realizes.

# Contained entries
| Level | Entry id | What it represents (one line) |
|-------|----------|-------------------------------|
| — | — | none — MOD-schema owns its forward scripts directly via `source:` (§2 exception) |

# Boundaries & dependencies
**Depends on** — | `MOD-model` | the `ENT-*` specs the forward schema scripts derive from |
**Exposes to** — | `MOD-api`/`MOD-web` | the applied schema |

# Conventions specific to the module
- **Schema is derived, not authored:** each `Vn` script cites the `ENT-*` it realizes; shipped scripts are append-only.
```
</example>
