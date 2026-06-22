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
| Test framework(s) | `<JUnit | xUnit | Vitest/Jest | pytest | go test>` (unit) + `<…>` (integration) + `<…>` (UI) |
| DB / persistence  | `<Postgres | MySQL | SQLite | none>` + `<migration tool>` |

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
| migrations           | owned by `MOD-build`, derived from entities | `db/migrations/<n>_<name>.sql`        |

- File comment syntax for the traceability header: `<// …>`.
- Design-token names referenced by UI specs resolve here: `<token source / theme file>`.

---

## 3. Canonical commands

The `test-runner` and `code-gatekeeper` use exactly these (installing deps as
needed). Fill them for the stack:

```
install:   <e.g. pnpm install>
build:     <e.g. pnpm build>
test-unit: <e.g. pnpm test:unit>
test-int:  <e.g. pnpm test:int>
test-all:  <e.g. pnpm test>
lint:      <e.g. pnpm lint>
run:       <e.g. pnpm dev>
migrate:   <e.g. pnpm migrate>
```

- Where the test framework supports it, **bake a machine-readable reporter into the
  `test-*` commands** (JSON / TAP / JUnit-XML) so `tests/REPORT.md` is parsed
  deterministically. The `test-runner` runs these commands **verbatim** and never
  adds or alters flags — a parseable reporter belongs here, in the canonical command.

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
