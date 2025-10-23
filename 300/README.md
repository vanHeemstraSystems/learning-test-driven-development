# 300 - Learning Our Subject

I’ll help you get started with professional-level TDD in Python for CI/CD environments. Here’s a comprehensive guide:

## 100 - Essential Books

See [README.md](./100/README.md)

## 200 - GitHub Repositories

See [README.md](./200/README.md)

## 300 - Pluralsight Courses

See [README.md](./300/README.md)

## 400 - Testing Frameworks & Tools for Enterprise TDD

See [README.md](./400/README.md)

## 500 - Enterprise TDD Best Practices

See [README.md](./500/README.md)

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

## 600 - Learning Path

See [README.md](./600/README.md)

1. Start with pytest basics and TDD fundamentals
1. Practice the Red-Green-Refactor cycle on small projects
1. Learn mocking and test doubles
1. Implement tests in a CI/CD pipeline
1. Study enterprise patterns (repository pattern, dependency injection)
1. Practice refactoring legacy code with tests

Would you like me to search for the most current Pluralsight courses or any newer resources that might have been published recently?​​​​​​​​​​​​​​​​
