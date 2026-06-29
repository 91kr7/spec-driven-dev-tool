# SCoT — Structured Chain-of-Thought (canonical grammar)

<instruction>Canonical contract. The *only* pseudo-code grammar allowed in behavioral specs. Language- and framework-agnostic: describes behavior (sequence / branch / loop, explicit I/O, invariants) — never library choices, API signatures, or syntax (these live in code + `.sdd/impl-notes/<MOD-id>/<level>/<id>.impl-notes.md`).</instruction>

## 1. Scope & purpose
<instruction>Used by `kind: behavioral` specs (`service`, `controller`, `use-case`). Not for structural specs or `gui` specs (→ `ui-schema.md`). A `gui` spec MAY embed a small SCoT snippet. Why a fixed grammar: faithful translation to any language; stable branch ids make coverage mechanical; equivalence judged by tests, not text diff.</instruction>

## 2. Lexical conventions
| Element | Notation |
|---|---|
| Comment | `# free text` |
| Assignment | `x <- expression` |
| Call | `CALL receiver.method(args)` / `r <- CALL …` |
| Reference | other specs by **id**: `CALL CLS-userRepo.save(user)` |
| Literal | `"text"`, `42`, `true`, `null` |
| Field access | `user.email` |

<instruction>Indentation = 2 spaces; `END` is authoritative for block scope. Neutral types (map via `target.md` §2): `Int Long Float Decimal Bool String Char Bytes Date DateTime UUID`, `List<T> Set<T> Map<K,V> Option<T> Result<T,E> Void Any`. Domain/structural types referenced by id/name (`User` = ENT-user).</instruction>

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
<instruction>`ASYNC FUNCTION` + `AWAIT` at call sites; `PURE FUNCTION` marks side-effect-free. Error style declared in front-matter `error_style: result|raise` (§6); body MAY restate it.</instruction>

## 4. Control constructs
<instruction>Every branching construct carries a branch id `[Bn]`. Plain sequence does not.</instruction>

```
[B1] IF <cond> THEN          # arm B1.then
ELSE IF <cond2> THEN         # arm B1.elif1
ELSE                         # arm B1.else
END
```
<instruction>No `ELSE`? Implicit fall-through is still arm `B1.else`.</instruction>

```
[B2] SWITCH <expr>
  CASE <label>:   …          # arm B2.case:<label>
  DEFAULT:        …          # arm B2.default
END

[B3] FOR EACH item IN coll   # arm B3.body (≥1 iter)
END                          # arm B3.empty (zero iters)

[B4] WHILE <cond>            # arm B4.body (entered ≥1)
END                          # arm B4.skip (false on entry)

[B5] REPEAT                  # arm B5.body (always ≥1)
UNTIL <cond>                 # arm B5.again (looped ≥2)
```
<instruction>`BREAK`/`CONTINUE` allowed in loops.</instruction>

```
[B6] TRY                     # arm B6.ok (no error)
CATCH e: <ErrorType>         # arm B6.catch:<ErrorType>
FINALLY                      # always runs; not a separate arm
END
```

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

<instruction>`LOG`/`EMIT`/`ASSERT` express intent; concrete mechanism goes in impl-notes.</instruction>

## 6. Error style — one per spec
<instruction>Declared in front-matter `error_style:` (conventions §3, canonical home):
- `result` — functions return `Result<T,E>` (`Ok`/`Err`).
- `raise` — functions `RAISE`; callers `TRY/CATCH`.
Implementer maps it to target idiom. Public interface table lists error cases regardless of style.</instruction>

## 7. Branch-id rules (coverage contract)
<rules>
1. Branch ids are unique within a `FUNCTION`, assigned top-to-bottom, stable across re-writes.
2. A branch's arms (§4) are its coverage set.
3. Canonical coverage id (used everywhere):
   - branch arm → `<spec-id>::<function>#<arm-id>` (e.g. `CLS-regCtrl::register#B1.else`)
   - acceptance criterion → `<spec-id>#ACn` (e.g. `CLS-regCtrl#AC2`)
4. test-writer creates ≥1 test per arm id and per `ACn`; test-gatekeeper REJECTs any uncovered.
5. Nested branches use flat function-level numbering; a nested arm is reached only through its parent's path.
</rules>

## 8. Worked example
<example name="CLS-regCtrl.register, error_style: result">
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
</example>

## 9. SCoT must NOT contain
<rules>
- Concrete library/framework names (`Spring`, `Express`, `axios`…).
- Language syntax (`@Annotations`, `public static`, decorators).
- SQL/ORM specifics — persistence is `CALL repo.method(...)`; schema derives from the entity spec.
- Exact third-party API signatures — express call intent.
Anything needing a concrete mechanism is a concretization → `.sdd/impl-notes/<MOD-id>/<level>/<id>.impl-notes.md`, never the spec.
</rules>
