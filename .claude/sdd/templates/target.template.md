# Target — stack, architecture & canonical commands

> **Per-project file.** `plan-architect` (`/sdd-auto` step 3) writes this from the user prompt.
> Stack unstated ⇒ leave explicit `<…>` placeholders (never a silent default); `plan-gatekeeper`
> REJECTs on any placeholder · command escalates the stack question to the human. Existing project
> ⇒ reflect the established stack; a feature may only extend/override it. **Part of the source of
> truth** — every code-writing agent reads it. Replace every `<…>`; unused fields read `n/a`,
> never a bare `<…>`.

## 1. Stack

| Aspect | Value |
|--------|-------|
| Project type | `<new app \| feature on existing SDD project>` |
| Architecture | `<monolith \| modular \| microservices \| library \| CLI>` |
| Primary language | `<Java \| Kotlin \| C# \| TypeScript \| Python \| Go \| …>` + version |
| Runtime | `<JVM 21 \| .NET 8 \| Node 20 \| Python 3.12 \| …>` |
| Backend framework | `<Spring \| ASP.NET \| Express/Nest \| FastAPI \| Gin \| none>` |
| Frontend framework | `<React \| Angular \| Vue \| Svelte \| none>` (≠ `none` ⇒ GUI project) |
| Build tool / package manager | `<Gradle \| Maven \| dotnet \| npm/pnpm \| uv/poetry \| go>` |
| Test framework(s) | `<unit>` + `<integration>` + `<component/UI>` + `<Playwright \| Cypress \| none>` (e2e — GUI only) |
| DB / persistence | `<Postgres \| MySQL \| SQLite \| none>` + `<schema-versioning tool, e.g. Flyway \| Prisma Migrate \| EF Core>` |

## 2. Source mapping, naming & language idioms

Each spec `kind` → path (used to propose `source:` for new specs):

| Spec kind | Path convention | Example |
|-----------|-----------------|---------|
| `service`/`controller` | `src/<module>/<Name>.<ext>` | `src/api/RegistrationController.ts` |
| `use-case` coordinator | `src/<module>/<Name>.<ext>` | `src/app/RegisterUser.ts` |
| `entity` | `src/<module>/model/<Name>.<ext>` | `src/db/model/User.ts` |
| `dto`/`enum` | `src/<module>/types/<Name>.<ext>` | `src/api/types/RegisterCmd.ts` |
| `interface`/`config` | `src/<module>/<Name>.<ext>` | `src/db/UserRepository.ts` |
| `gui` screen | `src/<module>/screens/<Name>.<ext>` | `src/web/screens/RegisterScreen.tsx` |
| `COMP-*` | `src/ui/<layer>/<Name>.<ext>` | `src/ui/atoms/Button.tsx` |
| schema changes | owned by `MOD-schema`, derived from entities | `db/schema/<n>_<name>.sql` |

- Traceability-header comment syntax: `<// …>`. Design-token source: `<theme/token file>`.
- **Test layout (one target per spec id, so a scoped run resolves mechanically):** `<unit/component → tests/unit/<id>.* ; integration → tests/integration/<FEAT-id>.* ; constraint → tests/model/<ENT-id>.* ; e2e → tests/e2e/<CLS-screen-id>.spec.*>`. Name each e2e file after the **screen id** (`CLS-*` gui), not the feature. Render `<id>` in a **filename/identifier-legal form** for the language (strip separators / PascalCase — e.g. `MOD-build` → `MODBuild`); the exact spec id survives verbatim only in each test's coverage-id comment (the matching source of truth), never forced into an illegal filename, never an empty "naming-stub" file.

### Language idioms — neutral-type → concrete calling convention

Specs/SCoT stay concretization-free (scot.md [§2](../scot.md#2-lexical-conventions)); the implementer **and** the test-writer BOTH derive the concrete callable form from this single map — so a spec-derived test compiles and calls the real code **without reading `src/`**. Fill this map up front: the contract that makes both sides converge instead of guessing.

| Neutral form (specs/SCoT) | Concrete idiom for this stack | Example call |
|---|---|---|
| entity / `dto` / `enum` type | `<record \| data class \| POJO+getters \| plain object>` | `<new Credential(u,v,a,s)>` |
| field accessor | `<x() \| getX() \| .x>` | `<cred.username()>` |
| construction | `<canonical ctor \| static factory \| builder \| setters>` | `<new LoginRequest(u,p)>` |
| `error_style: result` → `Result<T,E>` | `<concrete result type + ok/err builders>` | `<Result.ok(x) \| Result.err(e)>` |
| `error_style: raise` | `<exception types + how thrown>` | `<throw new BlankUsername()>` |
| `Option<T>` | `<Optional<T> \| nullable \| …>` | `<Optional.empty()>` |
| controller / HTTP boundary return | `<neutral Result rendered as a framework HTTP type? + the Outcome→status map>` | `<ResponseEntity<LoginResponse>, 200/400/401/500>` |

The implementer MUST follow these idioms (never free-style its own); the test-writer derives every call site / accessor / constructor from them. **A deviation from this map is a `code` defect, never a test defect.**

## 3. Canonical commands

`test-runner` and `code-gatekeeper` use exactly these (the runner fills only `{scope}`).

```
install:   <e.g. pnpm install>          # GUI: also `pnpm playwright install --with-deps`
build:     <e.g. pnpm build>
test-unit: <e.g. pnpm vitest run {scope} --reporter=dot --reporter=junit --outputFile=…>   # unit + component (mocked)
test-int:  <e.g. pnpm vitest run {scope} --reporter=dot --reporter=junit --outputFile=…>   # integration, infra mocked
test-e2e:  <e.g. pnpm playwright test {scope} --reporter=dot,junit>   # GUI ONLY — launches the running app; `n/a` otherwise
test-all:  <e.g. pnpm test>             # whole suite, UNSCOPED — the final arbiter run
lint:      <e.g. pnpm lint>
run:       <e.g. pnpm dev>
db-schema: <e.g. pnpm prisma migrate deploy>   # applies the entity-derived schema changes
```

- **Path discipline in `cd`-ing commands:** if a command changes into a build-unit dir (`cd frontend && …`), every path in it (output files, configs, selectors) is **relative to that dir or absolute** — never re-prefix the dir name. `cd frontend && … --outputFile=frontend/test-results/…` writes to `frontend/frontend/test-results/…`. One working dir, one path origin.
- **`{scope}`** — put the token where the framework's test selector goes; the runner replaces it with the in-scope selector (file globs or an id-alternation `--grep`/`--tests`). Empty `{scope}` = whole suite. The runner only fills this token — never adds or alters any other flag.
- **Two reporters baked in:** a compact **`dot`** console reporter (tiny captured output) **and** a machine-readable **file** reporter (JUnit-XML/JSON/TAP) the runner parses with `xmllint`/`jq`. Both belong here, in the canonical command — never added ad-hoc.

## 4. Notes / constraints
`<Non-functional constraints expressible as testable ACs (performance budgets, security, accessibility level). Anything not testable is context only and does not gate.>`
