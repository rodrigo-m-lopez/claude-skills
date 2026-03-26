---
name: tdd
description: >
  Test-Driven Development process for writing production code. Use this skill whenever
  implementing a new feature, fixing a bug, or modifying any existing production code.
  Apply it even for small changes — the discipline of writing the test first is what
  prevents regressions and keeps the codebase trustworthy. If you're about to write
  or change production code and haven't written a failing test yet, load this skill.
---

# Test-Driven Development

The core idea: production code only exists because a test demanded it. Writing the test first forces you to think about the desired behavior before the implementation, and it gives you a safety net for every future change.

---

## New feature

1. **Write the test(s) first.**
   Express the expected behavior as a test that fails. Use `@Test` with `assertThat`, `assertEquals`, etc. Write no production code yet — a failing test is the goal of this step.

2. **Confirm the test fails for the right reason.**
   Run only the new test. It should fail because the behavior doesn't exist yet, not because of a compilation error or misconfigured setup. A test that fails for the wrong reason is hiding a problem.

3. **Write the minimum code to make it pass.**
   Resist the urge to over-engineer. The simplest implementation that turns the test green is the right one at this stage.

4. **Confirm all tests pass** (regression check).

5. **Refactor with confidence**, keeping tests green throughout.

---

## Bug fix

1. **Reproduce the bug with a test.**
   Write a test that demonstrates the current wrong behavior — it must fail. A test that passes before any fix is not a useful test; revise it until it actually exposes the bug.

2. **Confirm the test fails for the right reason.**

3. **Fix the code** to make the test pass.

4. **Confirm all tests pass** (regression check).

---

## Core rules

- A test that passes before any implementation exists is useless — revisit it.
- Tests should verify observable behavior, not implementation details. Testing internals makes refactoring painful.
- Production code must have a corresponding test that required its existence.

---

## Test types

| Type | Scope | Characteristics |
|---|---|---|
| **Unit** | Single class/function | No I/O, no framework, uses builders/factories to set up state |
| **Integration** | Multiple components + framework | Real context (e.g. `@SpringBootTest`), external dependencies mocked (e.g. `@MockBean`) |
| **E2E** | Full system | Hits real dependencies; should not run in CI by default |

---

## Reference commands

Adapt to the project's build tool:

```bash
# Run a single test class (fast feedback loop)
./mvnw test -Dtest=MyClassName          # Maven
./gradlew test --tests MyClassName      # Gradle

# Run all tests (regression)
./mvnw test
./gradlew test
```
