# SCoT — Structured Chain-of-Thought (canonical grammar)

> **Status: canonical contract. Authored once, used by every agent.**
> This file defines the *only* pseudo-code grammar allowed in behavioral specs.
> It is **language- and framework-agnostic**: SCoT describes **behavior**
> (sequence / branch / loop, explicit inputs/outputs, invariants), never library
> choices, API signatures, or language syntax. Those belong in the code and in
> `.sdd/impl-notes/<spec-id>.md` — never in the spec.

SCoT is used by specs whose `kind:` is **behavioral** (`service`, `controller`,
`use-case`). It is **not** used for structural specs (`entity`, `dto`, `enum`,
`interface`, `config` → declarative) or for `gui` specs (→ `.sdd/ui-schema.md`).
A `gui` spec MAY embed a small SCoT snippet for a non-trivial event handler.

---

## 1. Why a fixed grammar

- **Faithful translation.** Restricting logic to *sequence / branch / loop*
  (plus `async`/`await` and `try`/`catch`) means the SCoT maps faithfully — not
  necessarily 1:1 — to Java, Kotlin, C#, TypeScript, Python, Go, etc.
- **Stable coverage.** Every decision point carries a **stable branch id**
  (`[B1]`, `[B2]`, …). The test-writer must produce at least one test per
  **branch arm** (`B1.then`, `B1.else`, …). Stable ids let coverage be checked
  mechanically and survive small edits to the surrounding text.
- **Behavioral equivalence.** Because the grammar omits implementation detail,
  a regenerated file is judged equivalent by the **tests** (behavioral), never by
  a textual diff.

---

## 2. Lexical conventions

| Element        | Notation                                              |
|----------------|-------------------------------------------------------|
| Comment        | `# free text to end of line`                          |
| Assignment     | `x <- expression`                                     |
| Call           | `CALL receiver.method(arg1, arg2)`                    |
| Call w/ result | `result <- CALL receiver.method(args)`                |
| Reference      | other specs by **id**: `CALL CLS-userRepo.save(user)` |
| Literal        | `"text"`, `42`, `3.14`, `true`, `false`, `null`       |
| Field access   | `user.email`                                          |
| Grouping       | `( … )` for expression precedence                     |

Indentation is **two spaces** per nested block and is for readability only — the
`END` keyword is authoritative for block scope.

### Type names (neutral)

Use neutral type names; the implementer maps them to the target language via
`.sdd/target.md`.

```
Int  Long  Float  Decimal  Bool  String  Char  Bytes  Date  DateTime  UUID
List<T>  Set<T>  Map<K,V>  Option<T>  Result<T,E>  Void  Any
```

Domain and structural types are referenced by **spec id or name**:
`User` (ENT-user), `RegisterCmd` (a DTO), `UserRepository` (CLS-userRepo).

---

## 3. Function / method header

Every behavioral unit is a `FUNCTION` with an explicit signature and explicit
input/output. Annotations sit between the header and the body.

```
FUNCTION <name>(<param>: <Type>, …) -> <ReturnType>
  INPUT:         <named inputs and their meaning, if not obvious>
  OUTPUT:        <what the return value means>
  PRECONDITION:  <what must hold on entry>           # optional
  POSTCONDITION: <what holds on normal exit>         # optional
  INVARIANT:     <what holds throughout>             # optional
  <body>
END
```

- `ASYNC FUNCTION …` marks an inherently asynchronous unit; use `AWAIT` at call
  sites: `user <- AWAIT CALL CLS-userRepo.save(user)`.
- A `PURE FUNCTION` annotation (optional) marks a side-effect-free function.
- Errors are expressed **either** as a `Result<T,E>` return **or** via `RAISE`
  (see §6). Pick one style **per spec** and state it; the implementer maps it to
  the target language's idiom (exceptions, error returns, `Either`, …).

---

## 4. Control constructs

Only the constructs below are allowed. **Every construct that branches carries a
branch id `[Bn]`.** Plain sequence (assignments, calls, `RETURN`) does **not**.

### 4.1 Sequence

```
x <- CALL service.load(id)
y <- x.value + 1
RETURN y
```

### 4.2 Two-way / multi-way branch — `IF`

```
[B1] IF <cond> THEN
  …                      # arm id: B1.then
ELSE IF <cond2> THEN
  …                      # arm id: B1.elif1
ELSE
  …                      # arm id: B1.else
END
```

Arm ids: `B1.then`, `B1.elif1`, `B1.elif2`, …, `B1.else`. If there is no `ELSE`,
the **implicit fall-through** is still an arm and is named `B1.else` for coverage
(the test-writer must exercise the path where no condition matched).

### 4.3 Selection — `SWITCH`

```
[B2] SWITCH <expr>
  CASE <value-or-label>:    …    # arm id: B2.case:<label>
  CASE <value-or-label>:    …    # arm id: B2.case:<label>
  DEFAULT:                  …    # arm id: B2.default
END
```

Use short stable `<label>`s (e.g. `CASE "ADMIN":` → `B2.case:ADMIN`).

### 4.4 Loops — `FOR`, `WHILE`, `REPEAT`

```
[B3] FOR EACH item IN collection
  …                        # arm id: B3.body  (≥1 iteration)
END
# arm id: B3.empty  (collection was empty — zero iterations)
```

```
[B4] WHILE <cond>
  …                        # arm id: B4.body  (entered ≥1 time)
END
# arm id: B4.skip  (cond false on entry — never entered)
```

```
[B5] REPEAT
  …                        # arm id: B5.body  (always ≥1 time)
UNTIL <cond>               # arm id: B5.again (looped ≥2 times)
```

Loop arms force the test-writer to cover **both** the iterating and the
boundary (empty / skip) paths. `BREAK` and `CONTINUE` are allowed inside loops.

### 4.5 Error handling — `TRY` / `CATCH` / `FINALLY`

```
[B6] TRY
  …                        # arm id: B6.ok      (no error raised)
CATCH e: <ErrorType>
  …                        # arm id: B6.catch:<ErrorType>
CATCH e: <OtherError>
  …                        # arm id: B6.catch:<OtherError>
FINALLY
  …                        # always runs; not a separate arm
END
```

---

## 5. Statements

| Statement | Meaning                                                              |
|-----------|---------------------------------------------------------------------|
| `RETURN e`         | return value `e` (or `RETURN` for `Void`)                  |
| `RAISE Error(msg)` | raise/throw an error (exception style)                     |
| `RETURN Ok(v)` / `RETURN Err(e)` | result style (matches a `Result<T,E>` return) |
| `BREAK` / `CONTINUE`             | loop control                                |
| `LOG <level> "msg"`              | structured logging intent (info/warn/error)|
| `EMIT <Event>(payload)`          | publish a domain event / message            |
| `ASSERT <cond>`                  | an invariant the code must enforce          |

`LOG`, `EMIT`, and `ASSERT` express **intent**; the implementer chooses the
concrete logger/bus/assertion mechanism and records it in impl-notes.

---

## 6. Error style — declare one per spec

State, near the top of the spec body, which error style the SCoT uses:

- **`error-style: result`** — functions return `Result<T,E>`; use `Ok`/`Err`.
- **`error-style: raise`** — functions `RAISE Error(...)`; callers `TRY/CATCH`.

The implementer maps the chosen style to the target language (e.g. `raise` →
exceptions in Java/Python, `result` → `Either`/sealed result in Kotlin/Rust-like,
Go `error` returns). The **public interface table** in the spec must list the
error cases regardless of style.

---

## 7. Branch-id rules (coverage contract)

1. Branch ids are **unique within a single `FUNCTION`** and assigned top-to-bottom
   (`[B1]`, `[B2]`, …). Re-running the spec writer must keep ids **stable**: never
   renumber existing branches; new branches take the next free number.
2. A branch's **arms** are enumerated by the rules in §4. The complete set of arm
   ids for a function is its **coverage set**.
3. The fully-qualified coverage id used by tests is
   `<spec-id>::<function>#<arm-id>` — e.g. `CLS-regCtrl::register#B1.else`.
4. The **test-writer** must create **at least one test per arm id** (and one per
   acceptance criterion `ACn`). The **test-gatekeeper** REJECTs if any arm or any
   `ACn` is uncovered.
5. Nested branches compose: a `[B2]` inside arm `B1.then` is still `[B2]` at the
   function level (flat numbering), and its arms (`B2.then`, …) are reached only
   on the `B1.then` path. Tests must reach the nested arm through its parent.

---

## 8. Worked example

`CLS-regCtrl` — `RegistrationController.register`, error-style: `result`.

```
error-style: result

FUNCTION register(cmd: RegisterCmd) -> Result<User, RegError>
  INPUT:  cmd  — { email: String, password: String, name: String }
  OUTPUT: Ok(User) on success; Err(RegError) on a rule violation
  PRECONDITION: cmd is non-null and structurally valid (DTO-validated)

  [B1] IF CALL CLS-userRepo.existsByEmail(cmd.email) THEN
    RETURN Err(EmailAlreadyTaken)          # arm B1.then
  ELSE
    # arm B1.else — email is free, continue
  END

  [B2] IF NOT CALL SHR-passwordPolicy.isStrong(cmd.password) THEN
    RETURN Err(WeakPassword)               # arm B2.then
  END
  # arm B2.else — password acceptable

  hash  <- CALL SHR-passwordHasher.hash(cmd.password)
  user  <- CALL ENT-user.new(cmd.email, cmd.name, hash)
  saved <- CALL CLS-userRepo.save(user)

  EMIT UserRegistered({ userId: saved.id, email: saved.email })
  LOG info "user registered"
  RETURN Ok(saved)
END
```

Coverage set for `register`: `B1.then`, `B1.else`, `B2.then`, `B2.else`.
Together with the spec's acceptance criteria (`AC1`, `AC2`, …) these are the
units the test-writer must cover.

---

## 9. What SCoT must NOT contain

- No concrete library or framework names (`Spring`, `Express`, `JPA`, `axios`…).
- No language syntax (`@Annotations`, `public static`, semicolons, decorators).
- No SQL / ORM specifics — persistence is expressed as `CALL repo.method(...)`;
  the schema/migration derives from the **entity** spec, not from SCoT.
- No exact API signatures of third-party calls — express the *intent* of the call.

If behavior cannot be expressed without naming a concrete mechanism, that detail
is a **concretization** and belongs in `.sdd/impl-notes/<spec-id>.md`, written by
the implementer — the gated spec stays clean and regenerable.
