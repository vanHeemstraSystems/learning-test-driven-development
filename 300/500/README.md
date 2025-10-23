# 500 - Enterprise TDD Best Practices

**Test Structure:**

- Use the Arrange-Act-Assert (AAA) pattern
- Follow the test pyramid (many unit tests, fewer integration tests, minimal E2E)
- Keep tests independent and idempotent
- Use fixtures for setup/teardown

**CI/CD Pipeline:**

```
1. Lint (flake8, black, mypy)
2. Unit tests (fast, isolated)
3. Integration tests
4. Coverage reporting (aim for 80%+ coverage)
5. Build artifacts
6. Deploy to staging/production
```

**Key Principles:**

- Write the test first (Red-Green-Refactor cycle)
- Keep tests fast (mock external dependencies)
- One assertion per test (when possible)
- Test behavior, not implementation
- Use descriptive test names
