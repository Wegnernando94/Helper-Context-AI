---
name: playwright-config
description: Playwright project configuration — defineConfig, projects, storageState, reporters, CI settings
---

# Playwright Config

## Project structure

```ts
export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [['html'], ['junit', { outputFile: 'results.xml' }], ['list']],

  use: {
    baseURL: process.env.BASE_URL ?? 'https://app.example.com',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    {
      name: 'setup',
      testMatch: '**/global.setup.ts',
    },
    {
      name: 'e2e',
      testIgnore: '**/global.setup.ts',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

## Rules

- `storageState` defined at project level — NEVER repeat `test.use({ storageState })` in individual specs
- `workers: 1` on CI prevents flakiness from shared database state
- `retries: 2` on CI only — locally retries hide real failures
- `fullyParallel: true` — each test file runs in its own worker
- `trace: 'on-first-retry'` — traces recorded only when test fails
- `baseURL` from env var with fallback to QA environment

## File layout

```
tests/
  global.setup.ts          # Login once, saves storageState
  fixtures/
    custom/                # Custom Playwright fixtures (app-fixtures.ts is the entry point)
    data/                  # Pure mock/data builder files (no Page dependency)
  <module>/
    <feature>/
      01-*.spec.ts
playwright/
  .auth/
    user.json              # Saved auth state (gitignored)
```
