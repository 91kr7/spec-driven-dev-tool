# SDD Conventions — single canonical reference

<instruction>Canonical contract. Every agent reads this first. On any conflict this file wins — except its two siblings it defers to: `scot.md` (behavioral grammar) and `ui-schema.md` (UI form).</instruction>

<values>
Two cross-cutting values bind every agent:
1. Markdown is the source of truth (authority). Specs decide WHAT the system does. On any conflict the spec wins and code is corrected, never the reverse. Code/tests are derived.
2. Reuse over repetition (DRY). Discover-before-create. Duplication above a small threshold is blocking.
</values>

## 1. File & folder layout
```
# THE TOOL — .claude/ (immutable; a project never edits it)
.claude/agents/*.md          # the 11 subagents
.claude/commands/sdd-auto.md # the single orchestrator command
.claude/sdd/conventions.md   # THIS FILE
.claude/sdd/scot.md          # behavioral grammar
.claude/sdd/ui-schema.md     # UI convention + reusable catalog
.claude/sdd/templates/*.md   # forms new specs are copied from

# THE PROJECT — SDD metadata: ALL under .sdd/
.sdd/target.md               # stack + canonical commands
.sdd/REQUIREMENT.md          # raw + refined requirement
.sdd/PLAN.md                 # plan output
.sdd/specs/modules.index.md       # GLOBAL skeleton: every MOD-*
.sdd/specs/<MOD-id>/<MOD-id>.spec.md    # module's own spec
.sdd/specs/<MOD-id>/<MOD-id>.index.md   # per-module roster
.sdd/specs/<MOD-id>/<level>/<id>.spec.md  # members; <level> ∈ {features,classes,model,ui-components,shared}
.sdd/specs/MOD-shared/{shared,ui-components}/<id>.spec.md  # cross-cutting home
.sdd/specs/REUSE-REPORT.md        # reuse-analyst output
.sdd/impl-notes/<MOD-id>/<level>/<id>.impl-notes.md  # concretization notes (exact mirror of specs/)
.sdd/verdicts/<nn>-<gate-agent>-<scope>-<verdict>.md  # append-only verdict log
.sdd/TEST-REPORT.md          # test result

# THE PROJECT — product: stays at root
src/                         # GENERATED source
tests/                       # GENERATED test files
```
<rules>
Organized by module, level inside. The `module:` front-matter is the folder, the id prefix is the level. Subfolders and `<MOD>.index.md` are created lazily. Cross-cutting placement (Rule A / LCA): an `SHR-*`/`COMP-*` whose consumers span ≥2 modules goes to `MOD-shared` (§13).
</rules>

## 2. Identifier scheme
| Prefix | Level / kind | Form | Example |
|---|---|---|---|
| `REQ-` | atomic requirement | `REQ-<nnn>` | `REQ-001` |
| `MOD-` | module | `MOD-<kebab>` | `MOD-api` |
| `FEAT-` | feature / use-case | `FEAT-<nnn>` | `FEAT-001` |
| `ENT-` | domain entity | `ENT-<kebab>` | `ENT-user` |
| `CLS-` | class / service / screen | `CLS-<lowerCamel>` | `CLS-userRepo` |
| `COMP-` | shared UI component | `COMP-<lowerCamel>` | `COMP-button` |
| `SHR-` | shared non-UI abstraction | `SHR-<lowerCamel>` | `SHR-passwordHasher` |
| `AC` | acceptance criterion | `AC<n>` | `AC1` |
| `B` | SCoT branch | `B<n>` + arm | `B1.then` |

<rules>
- Terminology: "entity" (generic) vs `ENT-` (specific domain entity).
- Ids are stable: never renumber/rename.
- `MOD-build` is mandatory. It owns build files/scaffolding. Carries no `depends_on`, every other module declares `depends_on: MOD-build` (always first slice). GUI project → owns e2e harness.
- `MOD-schema` is mandatory for a DB project with `ENT-*`. Owns DB schema changes derived from entity specs. Declares ≥1 script in `source:`. Ordered after entities. Not requirement-exempt: `requirements` = union of `ENT-*` REQs.
- `MOD-shared` is cross-cutting home (dependency SINK: depends on `MOD-build` only).
</rules>

## 3. Spec front-matter (YAML)
```yaml
---
id: CLS-regCtrl                 # required
name: RegistrationController    # required
kind: controller                # required
module: MOD-api                 # required
layer: organism                 # ui-components only
depends_on: [CLS-userRepo, ENT-user]   # topological order
requirements: [REQ-001]         # back-link id(s)
source: [src/api/RegistrationController.ts]   # AUTHORITATIVE spec→source mapping
owns_sections: []               # co-owned aggregator files
variants: [primary, secondary]  # ui-components only
error_style: result             # behavioral specs only
---
```
<rules>
- `status` lives ONLY in the index row.
- `source:` is single authoritative map. Propose paths from `target.md` for NEW.
- `requirements:` must be real ids or `—` for `MOD-build`. Shared/library specs → non-empty subset of consumer REQs (must be consumer-backed AND realized here). `MOD-schema` → union of `ENT-*` REQs. Empty or unbacked REQ = defect (orphan/excess).
- `error_style:` lives only in front-matter.
</rules>

### `kind:` → body form
| `kind:` | Category | Body form |
|---|---|---|
| `service`/`controller`/`use-case` | behavioral | SCoT (`scot.md`) |
| `entity` | structural | declarative — field table + relations + invariants |
| `dto`/`enum`/`config` | structural | declarative table |
| `interface` | structural | declarative — signatures only |
| `module` | structural | overview — Purpose, Contained entries, Boundaries |
| `gui` | ui | UI schematic (`ui-schema.md`) |

<rules>
A stub/mock is never specced — auto-derived from interface.
Required sections: Purpose · Public interface · Invariants & rules · body form · Acceptance criteria. `module` and `COMP-*` vary.
</rules>

### Acceptance-criterion altitudes
<rules>
- test-covered (untagged) — authored test asserts it.
- `(journey)` — crosses running stack; verified end-to-end by Playwright.
- `(pipeline)` — canonical `target.md` command success (install/build/run/migrate). Allowed ONLY on infra-module (`MOD-build`, `MOD-schema`). test-writer authors no test; test-gatekeeper counts covered from green result.
</rules>

## 4. Index rows — two index types
<rules>
Global `.sdd/specs/modules.index.md`:
| id | name | description | depends_on | spec | source | status |

Per-module `.sdd/specs/<MOD>/<MOD>.index.md`:
| id | name | description | level | depends_on | spec | source | status |
(module with `ui-component` appends `layer` + `variants`)

- Locate spec by id: `Glob .sdd/specs/**/<id>.spec.md`.
- `status` is canonical lifecycle home.
</rules>

## 5. Status lifecycle & separation of duties
<rules>
Per-entity status in index row: `draft → reviewed → implemented → approved`.
- Backward transition: spec change after `reviewed` → demote `→ draft`, fix, re-flow. Code/test bug never demotes spec.
- Separation of duties: Gatekeepers JUDGE (write `.sdd/verdicts/`); Command ADVANCES `status`; Authors WRITE artifacts.
</rules>

## 6. `.sdd/verdicts/` verdict records
<instruction>Append-only by file, never rewrite. Sorted filenames are the timeline.</instruction>
Path: `.sdd/verdicts/<nn>-<gate-agent>-<scope>-<verdict>.md`

```
## <ISO-8601> — <gate-agent> — <PASS|REJECT>
- scope: <ids reviewed>
- phase: <analysis|code|test>
- iteration: <n>/<budget>
- verdict: PASS | REJECT
- reasons:
  - <terse line per check>
- routing: <agent | escalate | none>
```
<rules>Verdict economy: record conclusion, not re-derivation. PASS = terse line per check group. REJECT = one line per blocking defect.</rules>

## 7. Iteration budgets & failure routing
| Loop | Budget | On overflow |
|---|---|---|
| analysis | 3 | escalate |
| code | 3 | escalate |
| test | 5 | escalate |

<rules>
Budgets are per scope.
test-gatekeeper routing:
- spec bug → `spec-writer`
- code bug → `code-implementer`
- test bug → `test-writer`
- build/setup failure → route by offending file or escalate.
- e2e-setup failure or phase skipped → escalate.
</rules>

## 8. Change policy
<rules>
- Edit (default): minimal diff to mapped files.
- Spec stays authority: edit brings code to spec. If spec wrong, fix spec first.
- Back-propagate to MD: concretizations go to `impl-notes`.
- Regenerate (exception): only when file is new or spec changed substantially.
- Equivalence = tests, not diff.
</rules>

## 9. Agent roster & isolation matrix
<instruction>11 subagents + 1 orchestrator command. Subagents are single-purpose, never spawn subagents.</instruction>
| Agent | Role | May WRITE | tools | Reads `src/`? | model |
|---|---|---|---|---|---|
| `requirement-analyst` | capture reqs | `.sdd/REQUIREMENT.md` | Read, Write, Edit, Glob, Grep | no | opus |
| `plan-architect` | plan + target | `.sdd/PLAN.md`, `target.md` | Read, Write, Edit, Glob, Grep | no | opus |
| `plan-gatekeeper` | judge plan | `.sdd/verdicts/` | Read, Write, Glob, Grep | no | opus |
| `spec-writer` | write specs | `.sdd/specs/` | Read, Write, Edit, Glob, Grep | no | opus |
| `reuse-analyst` | dedupe/promote | `.sdd/specs/` | Read, Write, Edit, Glob, Grep | no | opus |
| `analysis-gatekeeper` | judge specs | `.sdd/verdicts/` | Read, Write, Glob, Grep | no | opus |
| `code-implementer` | specs → source | `src/`, `impl-notes/` | Read, Write, Edit, Glob, Grep | yes (edit) | opus |
| `code-gatekeeper` | judge code | `.sdd/verdicts/` | Read, Write, Glob, Grep, Bash | yes (review) | opus |
| `test-writer` | specs → tests | `tests/` | Read, Write, Edit, Glob | no | sonnet |
| `test-runner` | run tests | `TEST-REPORT.md` | Read, Write, Glob, Bash | yes | sonnet |
| `test-gatekeeper` | coverage/triage | `.sdd/verdicts/` | Read, Write, Glob, Grep | yes | opus |

<rules>test-runner executes suite (canonical commands). test-writer NEVER reads `src/` or `impl-notes/`.</rules>

## 10. Command roster
| Command | Drives |
|---|---|
| `/sdd-auto` | Flow end-to-end, one vertical slice at a time. |

## 11. ROLE header (mandatory)
```
ROLE: <one-line identity>
MISSION: <outcome>
MINDSET: <MUST include: Markdown is source of truth; reuse over repetition (DRY)>
NON-GOALS: <never do>
```

## 12. Topological processing & vertical slices
<rules>
Process in `depends_on` topological order. One vertical slice at a time. Scaffolding first (`MOD-build` always first).
</rules>

## 13. Traceability
<rules>
- Shared nodes list subset of consumers' REQs.
- Rule A / LCA: Shared node lives in single module if all consumers there, else `MOD-shared`.
- Every generated source carries header: `// spec: CLS-regCtrl RegistrationController — .sdd/specs/MOD-api/classes/CLS-regCtrl.spec.md`. Comment-less formats exempt.
- Header is ONLY mandatory comment. Code must not re-narrate spec.
</rules>

## 14. `.sdd/TEST-REPORT.md` format
<instruction>test-runner writes; test-gatekeeper parses.</instruction>
```
# Test Report
## Run
- timestamp: <ISO-8601>
- scope: <whole-suite | in-scope ids>
- suites: <which ran>
- commands: install=<cmd> | build=<cmd> | unit=<cmd> | int=<cmd> | e2e=<cmd>
- exit-status: <0 | code>
- phase-reached: <install | build | unit | integration | component | e2e-setup | e2e | complete>
- tooling: <notes>

## Summary
- total: <n>
- passed: <n>
- failed: <n>
- skipped: <n>

## Failures
### <test name>
- coverage: <canonical id(s)>
- message: <msg>
- excerpt: |
    <trim>
```
