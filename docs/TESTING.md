# TESTING.md

> Testing facts for this codebase.

## Test Organization

### Location

Tests exist only within `recipes/vercel-neon-telegram`. All other modules in this monorepo (extensions, schemas, dashboards, integrations, skills, other recipes) do not have automated tests.

The 3 test files are co-located alongside source code in a `__tests__` subfolder:

```
recipes/vercel-neon-telegram/
└── src/
    └── lib/
        └── __tests__/
            ├── auth.test.ts        # Tests for src/lib/auth.ts
            ├── rate-limit.test.ts  # Tests for src/lib/rate-limit.ts
            └── types.test.ts       # Tests for src/lib/types.ts
```

### Naming Convention

Test files follow the `*.test.ts` pattern and are placed in a `__tests__/` directory adjacent to the source files they test.

## Frameworks Used

| Framework | Version | Purpose | Scope |
|-----------|---------|---------|-------|
| vitest | ^4.1.0 | Unit test runner | `recipes/vercel-neon-telegram` only |

No other test frameworks are declared or used anywhere in this repo.

## Test Types Present

### Unit Tests

- Location: `recipes/vercel-neon-telegram/src/lib/__tests__/`
- Count: 3 test files
- Coverage: Core shared library utilities — authentication key extraction/validation, rate-limit window logic, and Zod schema validation for thought types and metadata

No integration tests, end-to-end tests, or other test categories are present in this repo.

## Running Tests

Tests are scoped to the `recipes/vercel-neon-telegram` subdirectory and must be run from within that directory.

### All Tests

```bash
cd recipes/vercel-neon-telegram
npm test
```

This runs `vitest run` (single-pass, non-watch mode).

### Watch Mode

```bash
cd recipes/vercel-neon-telegram
npx vitest
```

## Test Patterns

### Fixtures

No shared fixture files. Test data is declared inline within each `describe` block using `const` values.

### Mocking

Tests use vitest's built-in mock utilities:

- `vi.stubEnv` / `vi.unstubAllEnvs` — for stubbing environment variables (e.g., `BRAIN_ACCESS_KEY`)
- `vi.useFakeTimers` / `vi.useRealTimers` — for controlling time in rate-limit window tests
- `vi.resetModules` — to obtain fresh module instances per test and avoid shared in-memory state

### Data Setup

No external data setup required. All tests use self-contained in-memory inputs (Request objects, plain objects, string values).

## Coverage

No coverage tool is configured. The `vitest` installation does not include `@vitest/coverage-v8` or `@vitest/coverage-istanbul`, and no `coverage` script is defined in `package.json`.
