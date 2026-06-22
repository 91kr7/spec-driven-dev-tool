# Requirement

> Project-seed template. Copy this file, fill every section, keep it 100% English
> and valid Markdown. This file is the **only** home of the raw + refined user
> requirement and is authored/owned per `.sdd/conventions.md` §9 by `plan-architect`
> (read by every downstream agent). Authority lives in Markdown: this captured
> requirement is what `REQ-` ids back-link to via each spec's `requirements:`
> front-matter (`.sdd/conventions.md` §3, §13).
>
> Cross-cutting values (verbatim, binding):
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**
>
> Conventions referenced (do not restate): id scheme `.sdd/conventions.md` §2,
> traceability `.sdd/conventions.md` §13. `REQ-<nnn>` ids are **stable** — never
> renumber an existing id; a new requirement takes the next free number.

---

## Raw requirement

> The user's request, captured verbatim — unedited, ambiguities and all. Do not
> "fix" it here; refinement happens in the next section. One paragraph or a short
> quoted block.

<paste the user's words exactly as received>

## Refined requirement

> The disambiguated, testable restatement. Resolve pronouns, name the actors,
> state the observable outcome. Every claim here must be traceable to a `REQ-` id
> below. If a refinement required a human decision (e.g. an unstated rule), note
> the decision inline.

<one or two paragraphs of clarified intent>

## In-scope

> Bullet list of what this requirement explicitly covers. Each bullet should map
> to one or more `REQ-` ids.

- <capability 1>
- <capability 2>

## Out-of-scope

> Bullet list of what is explicitly excluded, to bound the plan. State the reason
> when it prevents a likely misread.

- <excluded item 1 — reason>
- <excluded item 2 — reason>

## Requirement ids

> The atomic, individually-testable requirements. `REQ-<nnn>` ids are stable and
> are the back-link target of each spec's `requirements:` front-matter.

| id      | requirement (WHAT, one line)                          | in-scope | priority |
|---------|-------------------------------------------------------|----------|----------|
| REQ-001 | <one atomic, testable requirement>                    | yes      | must     |
| REQ-002 | <one atomic, testable requirement>                    | yes      | should   |

## Acceptance at a glance

> One Given/When/Then per `REQ-` id — the coarse acceptance the plan and specs
> must satisfy. Fine-grained `ACn` criteria live in the individual specs
> (`.sdd/conventions.md` §3); these are the requirement-level checks.

| req     | Given / When / Then                                                                 |
|---------|-------------------------------------------------------------------------------------|
| REQ-001 | Given <precondition>, When <action>, Then <observable outcome>                      |
| REQ-002 | Given <precondition>, When <action>, Then <observable outcome>                      |

---

## Example (filled) — user registration

> A complete worked instance of the template above for the user-registration
> domain. Delete this section in a real project (or keep it as the seed example).

### Raw requirement

> "Users should be able to sign up with their email, a password and their name.
> Don't let two people use the same email and make sure passwords aren't weak.
> After they register we should send some kind of welcome thing."

### Refined requirement

A visitor can create an account by submitting an **email**, a **password**, and a
**display name**. The system rejects a registration whose email already belongs to
an existing account (case-insensitive match), and rejects a password that does not
meet the strength policy (decision: minimum 12 characters with at least one letter
and one digit — confirmed with the human, recorded here as authority). On a
successful registration the system persists the account with the password stored
only as a salted hash and publishes a `UserRegistered` domain event so a welcome
email can be sent asynchronously by a downstream consumer.

### In-scope

- Account creation from email + password + display name (REQ-001).
- Duplicate-email rejection, case-insensitive (REQ-002).
- Password strength enforcement (REQ-003).
- Storing the password only as a salted hash, never in plaintext (REQ-004).
- Emitting a `UserRegistered` event on success (REQ-005).

### Out-of-scope

- Sending the welcome email itself — a downstream consumer of `UserRegistered`
  owns delivery; this requirement only emits the event.
- Login / authentication and session management — separate requirement.
- Email-address verification / confirmation links — separate requirement.
- Social / OAuth sign-up — not requested.

### Requirement ids

| id      | requirement (WHAT, one line)                                         | in-scope | priority |
|---------|----------------------------------------------------------------------|----------|----------|
| REQ-001 | A visitor can register with email, password and display name         | yes      | must     |
| REQ-002 | Registration is rejected when the email already exists (case-insens.)| yes      | must     |
| REQ-003 | Registration is rejected when the password fails the strength policy | yes      | must     |
| REQ-004 | A registered password is persisted only as a salted hash             | yes      | must     |
| REQ-005 | A `UserRegistered` event is emitted on successful registration       | yes      | should   |

### Acceptance at a glance

| req     | Given / When / Then                                                                                          |
|---------|-------------------------------------------------------------------------------------------------------------|
| REQ-001 | Given a valid email, strong password and name, When the visitor registers, Then a new account is created    |
| REQ-002 | Given an email already in use, When the visitor registers with it, Then the request is rejected as duplicate|
| REQ-003 | Given a password below the strength policy, When the visitor registers, Then the request is rejected as weak|
| REQ-004 | Given a successful registration, When the account is stored, Then the password is a salted hash, not plaintext|
| REQ-005 | Given a successful registration, When the account is created, Then a `UserRegistered` event is emitted       |
