# Target — stack, architecture & canonical commands

> **Per-project file.** The `plan-architect` (in `/sdd-auto` Phase A) writes this from the
> user prompt. If the stack is unstated it leaves explicit `<…>` placeholders (never a silent
> default); the `plan-gatekeeper` REJECTs on any placeholder and the command escalates the stack
> question to the human. On an existing project it reflects the established stack; a feature may
> only extend/override it. **Part of the source of truth** — every code-writing agent reads it.
> Replace every `<…>`; unused fields read `n/a`, never a bare `<…>`.

## 1. Stack

| Aspect | Value |
|--------|-------|
| Project type | `<new app | feature on existing SDD project>` |
| Architecture | `<monolith | modular | microservices | library | CLI>` |
| Primary language | `<Java | Kotlin | C# | TypeScript | Python | Go | …>` + version |
| Runtime | `<JVM 21 | .NET 8 | Node 20 | Python 3.12 | …>` |
| Backend framework | `<Spring | ASP.NET | Express/Nest | FastAPI | Gin | none>` |
| Frontend framework | `<React | Angular | Vue | Svelte | none>` (≠ `none` ⇒ GUI project) |
| Build tool / package manager | `<Gradle | Maven | dotnet | npm/pnpm | uv/poetry | go>` |
| Test framework(s) | `<unit>` + `<integration>` + `<component/UI>` + `<Playwright | Cypress | none>` (e2e — GUI only) |
| DB / persistence | `<Postgres | MySQL | SQLite | none>` + `<schema-versioning tool, e.g. Flyway | Prisma Migrate | EF Core>` |

## 2. Source path & naming conventions

How each spec `kind` maps to a path (used to propose `source:` for new specs):

| Spec kind | Path convention | Example |
|-----------|-----------------|---------|
| `service`/`controller` | `src/<module>/<Name>.<ext>` | `src/api/RegistrationController.ts` |
| `use-case` coordinator | `src/<module>/<Name>.<ext>` | `src/app/RegisterUser.ts` |
| `entity` | `src/<module>/model/<Name>.<ext>` | `src/db/model/User.ts` |
| `dto`/`enum` | `src/<module>/types/<Name>.<ext>` | `src/api/types/RegisterCmd.ts` |
| `interface`/`config` | `src/<module>/<Name>.<ext>` | `src/db/UserRepository.ts` |
| `gui` screen | `src/<module>/screens/<Name>.<ext>` | `src/web/screens/RegisterScreen.tsx` |
| `COMP-*` | `src/ui/<layer>/<Name>.<ext>` | `src/ui/atoms/Button.tsx` |
| schema changes | owned by `MOD-build`, derived from entities | `db/schema/<n>_<name>.sql` |

- Traceability-header comment syntax: `<// …>`. Design-token source: `<theme/token file>`.
- **Test layout (one target per spec id, so a scoped run resolves mechanically):** `<unit/component → tests/unit/<id>.* ; integration → tests/integration/<FEAT-id>.* ; constraint → tests/model/<ENT-id>.* ; e2e → tests/e2e/<CLS-screen-id>.spec.*>`. Name each e2e file after the **screen id** (`CLS-*` gui), not the feature.

## 3. Canonical commands

The `test-runner` and `code-gatekeeper` use exactly these (the runner fills only `{scope}`).

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

- **`{scope}`** — put the token where the framework's test selector goes; the runner substitutes the in-scope selector (file globs or an id-alternation `--grep`/`--tests`). Empty `{scope}` = whole suite. The runner fills it only — never adds/alters any other flag.
- **Two reporters baked in:** a compact **`dot`** console reporter (tiny captured output) **and** a machine-readable **file** reporter (JUnit-XML/JSON/TAP) the runner parses with `xmllint`/`jq`. Both belong here, in the canonical command — never added ad-hoc.

## 4. Iteration budgets (optional override of conventions §7)
```
analysis: 3
code: 3
test: 5
```

## 5. Notes / constraints
`<Non-functional constraints expressible as testable ACs (performance budgets, security, accessibility level). Anything not testable is context only and does not gate.>`
