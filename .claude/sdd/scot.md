# SCoT — Structured Chain-of-Thought (canonical grammar)

> **Canonical contract.** The *only* pseudo-code grammar allowed in behavioral specs.
> Language- and framework-agnostic: it describes **behavior** (sequence / branch /
> loop, explicit I/O, invariants) — never library choices, API signatures, or syntax
> (those live in code + `impl-notes/<level>/<id>.impl-notes.md`).

## 1. Scope & purpose

Used by `kind:` **behavioral** specs (`service`, `controller`, `use-case`). Not for structural specs (declarative) or `gui` specs (→ `ui-schema.md`). A `gui` spec MAY embed a small SCoT snippet for a non-trivial handler.

**Why a fixed grammar:** faithful (not 1:1) translation to any language; **stable branch ids** make coverage mechanical and edit-resistant; equivalence is judged by **tests**, never by textual diff.

---

## 2. Lexical conventions

| Element | Notation |
|---|---|
| Comment | `# free text` |
| Assignment | `x <- expression` |
| Call | `CALL receiver.method(args)` / `r <- CALL …` |
| Reference | other specs by **id**: `CALL CLS-userRepo.save(user)` |
| Literal | `"text"`, `42`, `true`, `null` |
| Field access | `user.email` |

Indentation = 2 spaces (readability only); `END` is authoritative for block scope.

**Neutral types** (implementer **and** test-writer both map via `target.md` §2's language-idioms map — same map ⇒ converging call sites, so a spec-derived test compiles against the real code without reading `src/`): `Int Long Float Decimal Bool String Char Bytes Date DateTime UUID`, `List<T> Set<T> Map<K,V> Option<T> Result<T,E> Void Any`. Domain/structural types referenced by id/name (`User` = ENT-user).

---

## 3. Function header

```
FUNCTION <name>(<param>: <Type>, …) -> <ReturnType>
  INPUT:         <named inputs, if not obvious>
  OUTPUT:        <what the return means>
  PRECONDITION:  <holds on entry>      # optional
  POSTCONDITION: <holds on normal exit># optional
  INVARIANT:     <holds throughout>    # optional
  <body>
END
```
- `ASYNC FUNCTION` + `AWAIT` at call sites; `PURE FUNCTION` marks side-effect-free.
- Error style is **declared in front-matter** `error_style: result|raise` (§6); the body MAY restate it for readability.

---

## 4. Control constructs

**Every branching construct carries a branch id `[Bn]`.** Plain sequence does not.

```
[B1] IF <cond> THEN          # arm B1.then
ELSE IF <cond2> THEN         # arm B1.elif1
ELSE                         # arm B1.else
END
```
No `ELSE`? the implicit fall-through is still arm `B1.else` (must be tested).

```
[B2] SWITCH <expr>
  CASE <label>:   …          # arm B2.case:<label>
  DEFAULT:        …          # arm B2.default
END
```

```
[B3] FOR EACH item IN coll   # arm B3.body (≥1 iter)
END                          # arm B3.empty (zero iters)

[B4] WHILE <cond>            # arm B4.body (entered ≥1)
END                          # arm B4.skip (false on entry)

[B5] REPEAT                  # arm B5.body (always ≥1)
UNTIL <cond>                 # arm B5.again (looped ≥2)
```
`BREAK`/`CONTINUE` allowed in loops.

```
[B6] TRY                     # arm B6.ok (no error)
CATCH e: <ErrorType>         # arm B6.catch:<ErrorType>
FINALLY                      # always runs; not a separate arm
END
```

---

## 5. Statements

| Statement | Meaning |
|---|---|
| `RETURN e` / `RETURN` | return value (or Void) |
| `RAISE Error(msg)` | raise/throw (exception style) |
| `RETURN Ok(v)` / `RETURN Err(e)` | result style |
| `BREAK` / `CONTINUE` | loop control |
| `LOG <level> "msg"` | logging intent |
| `EMIT <Event>(payload)` | publish a domain event |
| `ASSERT <cond>` | an invariant the code must enforce |

`LOG`/`EMIT`/`ASSERT` express **intent**; the concrete mechanism goes in impl-notes.

---

## 6. Error style — one per spec

Declared in front-matter `error_style:` (conventions §3, the canonical home):
- **`result`** — functions return `Result<T,E>` (`Ok`/`Err`).
- **`raise`** — functions `RAISE`; callers `TRY/CATCH`.

The implementer maps it to the target idiom. The **Public interface** table lists error cases regardless of style.

---

## 7. Branch-id rules (coverage contract)

1. Branch ids are **unique within a `FUNCTION`**, assigned top-to-bottom, **stable** across re-writes (new branches take the next free number).
2. A branch's **arms** (§4) are its **coverage set**.
3. **§7.3 — canonical coverage id** (used everywhere, no other spelling):
   - branch arm → `<spec-id>::<function>#<arm-id>` (e.g. `CLS-regCtrl::register#B1.else`)
   - acceptance criterion → `<spec-id>#ACn` (e.g. `CLS-regCtrl#AC2`)
   The test-writer tags each test with it, the test-runner echoes it **verbatim**, the test-gatekeeper joins on it.
4. test-writer creates **≥1 test per arm id and per `ACn`**; test-gatekeeper REJECTs any uncovered.
5. Nested branches use flat function-level numbering; a nested arm is reached only through its parent's path.

---

## 8. Worked example — `CLS-regCtrl.register`, `error_style: result`

```
FUNCTION register(cmd: RegisterCmd) -> Result<User, RegError>
  PRECONDITION: cmd is non-null and DTO-valid

  [B1] IF CALL CLS-userRepo.existsByEmail(cmd.email) THEN
    RETURN Err(EmailAlreadyTaken)          # B1.then
  ELSE
    # B1.else — email free
  END

  [B2] IF NOT CALL SHR-passwordPolicy.isStrong(cmd.password) THEN
    RETURN Err(WeakPassword)               # B2.then
  END                                      # B2.else

  hash  <- CALL SHR-passwordHasher.hash(cmd.password)
  user  <- CALL ENT-user.new(cmd.email, cmd.name, hash)
  saved <- CALL CLS-userRepo.save(user)
  EMIT UserRegistered({ userId: saved.id })
  RETURN Ok(saved)
END
```
Coverage set: `B1.then`, `B1.else`, `B2.then`, `B2.else` + the spec's `ACn`.

---

## 9. SCoT must NOT contain
- Concrete library/framework names (`Spring`, `Express`, `axios`…).
- Language syntax (`@Annotations`, `public static`, decorators).
- SQL/ORM specifics — persistence is `CALL repo.method(...)`; schema derives from the **entity** spec.
- Exact third-party API signatures — express call *intent*.

Anything needing a concrete mechanism is a **concretization** → `impl-notes/<level>/<id>.impl-notes.md`, never the spec.
