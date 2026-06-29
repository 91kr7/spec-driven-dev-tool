<!--
<instructions>
TEMPLATE: Target — stack, architecture & canonical commands
Owned by `plan-architect`. Replace ALL `<…>` placeholders; unused fields read `n/a`.
Part of the source of truth — read by all code-writing agents.
</instructions>
-->
# Target — stack, architecture & canonical commands
<instruction>The plan-architect (step 3) writes this from the user prompt. Explicit `<…>` placeholders left if unstated (plan-gatekeeper REJECTs these). On existing projects, reflects established stack; features only extend/override it. Replace every `<…>`; unused fields read `n/a`.</instruction>

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
| DB / persistence | `<Postgres \| MySQL \| SQLite \| none>` + `<schema-versioning tool>` |

## 2. Source mapping, naming & language idioms
<instruction>How each spec `kind` maps to a path (used for `source:`).</instruction>

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
- **Test layout:** `<unit/component → tests/unit/<id>.* ; integration → tests/integration/<FEAT-id>.* ; constraint → tests/model/<ENT-id>.* ; e2e → tests/e2e/<CLS-screen-id>.spec.*>`. Name each e2e file after the screen id. Render `<id>` in filename-legal form.

### Language idioms — neutral-type → concrete calling convention
<instruction>Specs/SCoT stay concretization-free. Implementer AND test-writer derive concrete callable form from this map.</instruction>

| Neutral form (specs/SCoT) | Concrete idiom for this stack | Example call |
|---|---|---|
| entity / `dto` / `enum` type | `<record \| data class \| POJO+getters \| plain object>` | `<new Credential(u,v,a,s)>` |
| field accessor | `<x() \| getX() \| .x>` | `<cred.username()>` |
| construction | `<canonical ctor \| static factory \| builder \| setters>` | `<new LoginRequest(u,p)>` |
| `error_style: result` | `<concrete result type + ok/err builders>` | `<Result.ok(x) \| Result.err(e)>` |
| `error_style: raise` | `<exception types + how thrown>` | `<throw new BlankUsername()>` |
| `Option<T>` | `<Optional<T> \| nullable \| …>` | `<Optional.empty()>` |
| controller / HTTP | `<neutral Result rendered as HTTP type? + Outcome→status map>` | `<ResponseEntity<LoginResponse>, 200/400/401/500>` |

## 3. Canonical commands
<instruction>Used by test-runner and code-gatekeeper. test-runner fills only `{scope}`. Two reporters baked in: console (dot) AND file (JUnit-XML/JSON/TAP).</instruction>

```
install:   <e.g. pnpm install>          # GUI: also `pnpm playwright install --with-deps`
build:     <e.g. pnpm build>
test-unit: <e.g. pnpm vitest run {scope} --reporter=dot --reporter=junit --outputFile=…>
test-int:  <e.g. pnpm vitest run {scope} --reporter=dot --reporter=junit --outputFile=…>
test-e2e:  <e.g. pnpm playwright test {scope} --reporter=dot,junit>   # GUI ONLY; `n/a` otherwise
test-all:  <e.g. pnpm test>             # whole suite, UNSCOPED
lint:      <e.g. pnpm lint>
run:       <e.g. pnpm dev>
db-schema: <e.g. pnpm prisma migrate deploy>
```
- **Path discipline:** One working dir, one path origin.

## 4. Iteration budgets (optional override of conventions §7)
```
analysis: 3
code: 3
test: 5
```

## 5. Notes / constraints
<instruction>Non-functional constraints expressible as testable ACs. Anything not testable is context only and does not gate.</instruction>
`<constraint>`
