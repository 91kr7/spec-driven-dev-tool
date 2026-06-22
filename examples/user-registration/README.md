# Worked Example — User registration (Database + API)

> **⚠️ FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real `specs/`.**
> Everything under `examples/user-registration/` is a self-contained teaching
> slice that shows what a *complete, cross-referenced* set of SDD artifacts looks
> like for one feature. It is **spec-level only** — no `src/` or `tests/` are
> generated here. **Delete this whole directory before you start real work.**
> The example deliberately mirrors the canonical contracts in `.sdd/`
> (`conventions.md`, `scot.md`, `ui-schema.md`); if you fork it, keep obeying
> those, not this README.

Two cross-cutting values bind every artifact, here as everywhere:
**Markdown is the source of truth (authority); reuse over repetition (DRY).**

---

## What this example demonstrates

This is the **"User registration (Database + API)"** worked example: one feature
(`FEAT-001`) sliced cleanly across four modules, with every level of the SDD
spec hierarchy filled in and **cross-referenced by id** so the parts interlock.
It shows, end to end:

- a filled **target** (`.sdd/target.md`) pinning stack + canonical commands;
- one **feature** (purely compositional — orchestration only, `source: []`);
- one **entity** (`ENT-user`) from which both the TS type **and** the Postgres
  migration derive;
- three **classes** (a repository `interface`, a `controller` with full SCoT, a
  `gui` screen that composes UI components by id);
- one **shared** non-UI abstraction (`SHR-passwordHasher`);
- three **promoted UI components** (`COMP-textInput`, `COMP-button`,
  `COMP-formField`) — illustrating *discover-before-create* and progressive
  enrichment by the reuse-analyst;
- an **impl-note** (`CLS-regCtrl`) showing concretization that lives **outside**
  the gated spec (the test-writer never reads it).

---

## File map

```
examples/user-registration/
├── README.md                              # this banner
├── .sdd/
│   ├── target.md                          # stack + canonical build/test/run commands
│   └── impl-notes/
│       └── CLS-regCtrl.md                 # implementer-owned concretization (NOT gated)
└── specs/
    ├── indexes/
    │   ├── modules.index.md               # MOD-db, MOD-api, MOD-web, MOD-build
    │   ├── features.index.md              # FEAT-001
    │   ├── model.index.md                 # ENT-user
    │   ├── classes.index.md               # CLS-userRepo, CLS-regCtrl, CLS-registerScreen
    │   └── ui-components.index.md          # COMP-textInput, COMP-button, COMP-formField
    ├── modules/
    │   ├── MOD-db.spec.md                  # Persistence
    │   ├── MOD-api.spec.md                 # HTTP API
    │   ├── MOD-web.spec.md                 # Web UI
    │   └── MOD-build.spec.md               # Build/Infra (+ users migration)
    ├── features/
    │   └── FEAT-001.spec.md                # User registration (orchestration + integration ACs)
    ├── model/
    │   └── ENT-user.spec.md                # User entity (fields, invariants, constraint ACs)
    ├── classes/
    │   ├── CLS-userRepo.spec.md            # UserRepository (interface)
    │   ├── CLS-regCtrl.spec.md             # RegistrationController (SCoT)
    │   └── CLS-registerScreen.spec.md      # RegisterScreen (gui)
    ├── shared/
    │   └── SHR-passwordHasher.spec.md      # password hashing abstraction
    └── ui-components/
        ├── COMP-textInput.spec.md          # atom
        ├── COMP-button.spec.md             # atom
        └── COMP-formField.spec.md          # molecule
```

---

## How the artifacts cross-reference

Every spec back-links to a requirement and forward-links to its collaborators by
**id only** — so the traceability chain
**REQUIREMENT → FEATURE → CLASS → SOURCE → TEST** is reconstructable (see
`/sdd-trace` in `.sdd/conventions.md` §13).

```
REQ-001  (user registration requirement)
   │
   └── FEAT-001  "User registration"   (module MOD-api · kind use-case · source: [])
         │  orchestration SCoT:
         │  CLS-registerScreen.submit → CLS-regCtrl.register(cmd)
         │      → on Ok navigate /welcome · on Err show message
         │
         ├── CLS-registerScreen  (MOD-web · gui)  ── composes ──►
         │       COMP-formField ─► COMP-textInput        (×3 fields)
         │       COMP-button (secondary "Cancel", primary "Register")
         │       layout primitives COMP-appShell/header/body/footer/panel/stack
         │           ── from the SHARED SDD UI library (referenced, not redefined)
         │
         ├── CLS-regCtrl  (MOD-api · controller · error_style result) ── calls ──►
         │       CLS-userRepo.existsByEmail / .save
         │       SHR-passwordHasher.hash
         │       ENT-user.new
         │       └── concretization → .sdd/impl-notes/CLS-regCtrl.md
         │
         ├── CLS-userRepo  (MOD-db · interface)  ── depends_on ──►  ENT-user
         │       (a stub is auto-derived from this interface)
         │
         ├── ENT-user  (MOD-db · entity)
         │       └── DERIVES ──► the users-table migration in MOD-build
         │
         └── SHR-passwordHasher  (shared · interface)
                 (a stub is auto-derived from this interface)
```

`source:` front-matter is the **authoritative** spec→source map; each index's
`source` column is **derived** from it. `FEAT-001` is purely compositional, so
its `source` is `[]` — it only drives integration tests.

---

## How a learner would "run" this slice

This example stops at the **reviewed** spec level on purpose. To turn it into code
and tests you would (against a copy outside `examples/`):

1. **`/sdd-implement`** — the `code-implementer` walks the specs in `depends_on`
   order (`ENT-user` → `CLS-userRepo` → `SHR-passwordHasher` → `CLS-regCtrl` →
   `CLS-registerScreen`, plus `MOD-build`'s migration) and generates the files in
   each spec's `source:` list under `src/`, recording concretization in
   `.sdd/impl-notes/<id>.md`. The `code-gatekeeper` judges code ≡ spec.
2. **`/sdd-test`** — the `test-writer` derives an **independent** suite from the
   spec ACs and SCoT branch arms (it never reads `src/` or the impl-notes):
   unit tests from `CLS-*`, constraint tests from `ENT-user`, and integration
   tests from `FEAT-001`'s ACs (`AC1` new email → persisted + welcome; `AC2`
   duplicate → `EmailAlreadyTaken`, nothing persisted; `AC3` weak password →
   `WeakPassword`). On full green the `test-gatekeeper` PASSes and the slash
   command advances each entity's index `status` to `approved`.

Until then, every entity here sits at `status: reviewed` (spec vetted; code and
tests would follow).
