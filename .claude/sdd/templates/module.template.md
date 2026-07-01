<!--
  TEMPLATE — MODULE spec (kind: module, structural overview). Schema per conventions [§3](../conventions.md#s3).
  A module partitions the system; names the entries it contains — no SCoT, no UI schematic.
  Copy to .sdd/specs/<MOD-id>/<MOD-id>.spec.md (module spec sits at root of its own folder; register in global modules.index.md). Markdown is source of truth; reuse over repetition (DRY).
  Delete the "## Filled example" from a real spec.
-->
---
id: MOD-<kebab>                 # required — matches filename
name: <HumanModuleName>         # required
kind: module                   # required — drives the structural-overview FORM
module: MOD-<kebab>             # required — self-reference (a module is its own home)
depends_on: [MOD-<other>]      # other MODULE ids — topological; [] if none
requirements: [REQ-<nnn>]      # back-link; [] if purely infra
source: []                     # usually [] — EXCEPT MOD-build / MOD-schema own their infra FILES here (conventions §2)
owns_sections: []
---

# Purpose
<One paragraph: the cohesive responsibility this module owns + what falls outside it. Boundary-defining, not a feature list.>

# Contained entries
<Every id with `module: MOD-<this>` + a matching index row. Ids only, one line each. Omit empty groups.>

| Level | Entry id | What it represents (one line) |
|-------|----------|-------------------------------|
| Feature | `FEAT-<nnn>` | <use-case orchestrated here> |
| Entity | `ENT-<kebab>` | <domain entity owned here> |
| Class | `CLS-<lowerCamel>` | <service/controller/screen> |
| Component | `COMP-<lowerCamel>` | <shared UI component> |
| Shared | `SHR-<lowerCamel>` | <shared non-UI abstraction> |

# Boundaries & dependencies
<Must equal `depends_on:` + the union of contained entries' cross-module deps (no hidden coupling). Acyclic at module level ([§12](../conventions.md#s12)).>

**Depends on** — | `MOD-<other>` | <what it uses from it> |
**Exposes to** — | `MOD-<other>` | <entry ids other modules may reference> | (anything unlisted is internal)

# Conventions specific to the module
<Only rules special to THIS module, not already in conventions/target. Else: "None beyond conventions.md.">
- <module-local rule, or "None beyond conventions.md.">

---

## Filled examples — the infra modules `MOD-build` & `MOD-schema` (conventions [§2](../conventions.md#s2))

> Both are the documented exception to `source: []`: they own their infra **files directly**. `MOD-build` is pure scaffolding — **no domain `depends_on`** → first slice; a GUI project also lists `playwright.config.ts` and sets `target.md` `test-e2e` to a real command. `MOD-schema` (DB projects only) owns the forward schema scripts — each append-only, traced to an `ENT-*`; the code-implementer materializes the SQL from the entity tables.

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
The mandatory scaffolding module: build files, manifests, runtime config, CI, the app entry, and (GUI) the e2e harness config. Pure scaffolding — no domain dependency — so it builds first. Domain logic lives elsewhere and is merely built/shipped here.

# Contained entries
| Level | Entry id | What it represents (one line) |
|-------|----------|-------------------------------|
| — | — | none — MOD-build owns its infra files directly via `source:` ([§2](../conventions.md#s2) exception) |

# Boundaries & dependencies
**Depends on** — | — | nothing (pure scaffolding) |
**Exposes to** — | `MOD-api`/`MOD-web` | built artifacts (+ GUI: the e2e harness) |

# Conventions specific to the module
- **Config keys declared once** (key/type/default); no module reads an undeclared env var.
- **CI order fixed:** install → lint → build → test (incl. e2e for GUI) → package.
```

```markdown
---
id: MOD-schema
name: Database Schema
kind: module
module: MOD-schema
depends_on: [MOD-model, ENT-user]
requirements: [REQ-001]         # NOT exempt — the union of the REQ-* of the ENT-* it materializes
source: [db/schema/V1__create_user.sql]
owns_sections: []
---

# Purpose
The DB schema module (DB projects only): forward-only schema-change scripts **derived from the entity specs in MOD-model** (never hand-authored). Ordered after the entities it realizes.

# Contained entries
| Level | Entry id | What it represents (one line) |
|-------|----------|-------------------------------|
| — | — | none — MOD-schema owns its forward scripts directly via `source:` ([§2](../conventions.md#s2) exception) |

# Boundaries & dependencies
**Depends on** — | `MOD-model` | the `ENT-*` specs the forward schema scripts derive from |
**Exposes to** — | `MOD-api`/`MOD-web` | the applied schema |

# Conventions specific to the module
- **Schema is derived, not authored:** each `Vn` script cites the `ENT-*` it realizes; shipped scripts are append-only/immutable — a change adds the next `Vn`.
```
