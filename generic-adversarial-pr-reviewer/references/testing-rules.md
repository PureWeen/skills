# Testing Review Rules

Framework-agnostic testing guidance. Load when test files are changed (files
under `tests/`, `**/Tests/`, `**/*Test*`, `**/*Spec*`, or test project
directories).

---

## Testing Checks

| Check | What to look for |
|-------|-----------------|
| **Bug fixes need regression tests** | Every PR that fixes a bug should include a test that fails without the fix and passes with it. If the PR description says "fixes #N" but adds no test, ask for one. |
| **Test assertions must be specific** | `Assert.IsNotNull(result)` or `Assert.IsTrue(success)` don't tell you what went wrong. Prefer assertions with expected/actual values for richer failure messages. |
| **Deterministic test data** | Tests should not depend on system locale, timezone, or current date. Use explicit invariant culture and hardcoded dates when testing formatting. |
| **Test edge cases** | Empty collections, null inputs, boundary values, concurrent calls, and very large inputs should all be considered. If the PR only tests the happy path, suggest edge cases. |
| **Avoid over-mocking** | Not everything needs to be mocked. Integration tests catch real API changes that mocks never will. Mock at the boundaries, not inside the unit. |
| **Test isolation** | Tests should not depend on execution order or shared mutable state. Each test should set up its own preconditions and clean up after itself. |
| **Meaningful test names** | Test names should describe the scenario and expected outcome, not the method being tested. `Login_WithExpiredToken_ReturnsUnauthorized` is better than `TestLogin3`. |
