# Target — stack, architecture & canonical commands

> **TEMPLATE / per-project file.** The `plan-architect` (via `/sdd-plan`) **writes
> this file from the user prompt** at the start of a new project or feature. If the
> user prompt does not state the stack, the `plan-architect` leaves the unresolved
> fields as explicit `<…>` placeholders (it never silently assumes a default); the
> `plan-gatekeeper` REJECTs on any placeholder and the driving command escalates the
> stack question to the human — a subagent never prompts interactively. For an existing SDD project this file
> already reflects the established stack; a new feature may only extend/override it.
>
> This is a Markdown file and therefore **part of the source of truth**. Every
> code-writing agent reads it to know which language/framework to emit, where files
> go, and which commands to run. Replace every `<…>` placeholder.

---

## 1. Stack

| Aspect            | Value                                             |
|-------------------|---------------------------------------------------|
| Project type      | `<new app | feature on existing SDD project>`     |
| Architecture      | `<monolith | modular | microservices | library | CLI>` |
| Primary language  | `<Java | Kotlin | C# | TypeScript | Python | Go | …>` + version |
| Runtime           | `<JVM 21 | .NET 8 | Node 20 | Python 3.12 | …>`    |
| Backend framework | `<Spring Boot | Jakarta | ASP.NET | Express/Nest | FastAPI | Gin | none>` |
| Frontend framework| `<React | Angular | Vue | Svelte | none>`         |
| Build tool        | `<Gradle | Maven | dotnet | npm/pnpm | uv/poetry | go>` |
| Package manager   | `<…>`                                             |
| Test framework(s) | `<JUnit | xUnit | Vitest/Jest | pytest | go test>` (unit) + `<…>` (integration) + `<…>` (component/UI) + `<Playwright | Cypress | none>` (e2e — **GUI projects only**, drives the running app in a real browser) |
| DB / persistence  | `<Postgres | MySQL | SQLite | none>` + `<schema-versioning tool, e.g. Flyway | Liquibase | Alembic | Prisma Migrate | EF Core — what most stacks call "migrations">` |

---

## 2. Source path & naming conventions

How each spec `kind` maps to a file path and extension (used to propose `source:`
for new specs). Adapt to the stack:

| Spec kind            | Path convention                          | Example                                  |
|----------------------|------------------------------------------|------------------------------------------|
| `service`/`controller`| `src/<module>/<Name>.<ext>`             | `src/api/RegistrationController.ts`      |
| `use-case` coordinator| `src/<module>/<Name>.<ext>`             | `src/app/RegisterUser.ts`                |
| `entity`             | `src/<module>/model/<Name>.<ext>`        | `src/db/model/User.ts`                   |
| `dto`/`enum`         | `src/<module>/types/<Name>.<ext>`        | `src/api/types/RegisterCmd.ts`           |
| `interface`          | `src/<module>/<Name>.<ext>`              | `src/db/UserRepository.ts`               |
| `config`             | `<repo-root config path>`                | `src/config/app.config.ts`               |
| `gui` screen         | `src/<module>/screens/<Name>.<ext>`      | `src/web/screens/RegisterScreen.tsx`     |
| `COMP-*` component   | `src/ui/<layer>/<Name>.<ext>`            | `src/ui/atoms/Button.tsx`                |
| schema changes           | owned by `MOD-build`, derived from entities | `db/schema/<n>_<name>.sql`        |

- File comment syntax for the traceability header: `<// …>`.
- Design-token names referenced by UI specs resolve here: `<token source / theme file>`.
- **Test layout (one test target per spec id, so a scoped run resolves mechanically):**
  `<e.g. unit/component → tests/unit/<id>.* ; integration → tests/integration/<FEAT-id>.* ; constraint → tests/model/<ENT-id>.* ; e2e → tests/e2e/<CLS-screen-id>.spec.*>`. Each
  test file's name/path carries the spec id it covers so the `test-runner` can map scope ids → test files for the `{scope}` selector. **Name each e2e file after the screen id (`CLS-*` gui)** it drives — not the feature id — so a screen-scoped run resolves its own e2e (a feature's scope closure already includes its screens).

---

## 3. Canonical commands

The `test-runner` and `code-gatekeeper` use exactly these (installing deps as
needed). Fill them for the stack:

```
install:   <e.g. pnpm install>   # GUI projects: also provision the e2e browser binaries, e.g. `pnpm install && pnpm playwright install --with-deps`
build:     <e.g. pnpm build>
test-unit: <e.g. pnpm vitest run {scope}>      # unit (classes) + component (gui, mocked); {scope} = in-scope selector — see below
test-int:  <e.g. pnpm vitest run {scope}>      # integration (features), infrastructure mocked
test-e2e:  <e.g. pnpm playwright test {scope}> # GUI projects ONLY — real running app in a real browser; the command launches the app under test (e.g. Playwright `webServer`). `n/a` for backend/CLI/library projects.
test-all:  <e.g. pnpm test>                    # whole suite, UNSCOPED — the final arbiter run
lint:      <e.g. pnpm lint>
run:       <e.g. pnpm dev>
db-schema: <command that applies the entity-derived schema changes, e.g. ./gradlew flywayMigrate | pnpm prisma migrate deploy>
```

- **`{scope}` placeholder (scoped runs).** Write each `test-unit` / `test-int` /
  `test-e2e` command with a `{scope}` token where the framework's test selector goes.
  The `test-runner` substitutes the in-scope selector for the scope under work (test-file
  globs, or an id-alternation `--grep`/`--tests` pattern — whichever the command selects
  by) so **only the scope's tests run during the workflow, never the whole app**. An
  **empty `{scope}` means the whole suite** (used by `test-all` and the final unscoped
  arbiter run). Keep the token where a selector validly appears; the runner only fills it
  — it never adds or alters any other flag.
- **Two reporters: dot + structured.** Where the framework supports it, bake **both** into
  each `test-*` command — a compact **`dot`** console reporter (one char per test, so the
  agent's captured output stays tiny) **and** a machine-readable **file** reporter
  (JUnit-XML / JSON / TAP written to a file) that the `test-runner` parses deterministically
  (`xmllint`/`jq`) without reading a huge log. E.g. `vitest run {scope} --reporter=dot
  --reporter=junit --outputFile=…`; `playwright test {scope} --reporter=dot,junit`. The
  `test-runner` runs these commands **verbatim** — both reporters belong here, in the
  canonical command, never added ad-hoc by the runner.

---

## 4. Iteration budgets (optional override)

Override the defaults from `.claude/sdd/conventions.md` §7 if needed:

```
analysis: 3
code: 3
test: 5
```

---

## 5. Notes / constraints

`<Non-functional constraints expressible as testable acceptance criteria
(performance budgets, security rules, accessibility level). Anything not testable
is recorded here as context but does not gate.>`
