---
name: playwright-patterns
description: Test structure — naming, AAA with test.step(), Custom Fixtures, describe blocks, tags
---

# Playwright Patterns

## Naming convention

```
tests/<module>/<feature>/<NN>-<slug>.spec.ts
```

Examples:
- `tests/orders/report/01-initial-state.spec.ts`
- `tests/documents/purchase-orders/po-01-listing.spec.ts`

- `NN` = sequence number (01, 02...) — determines execution order within the feature
- `slug` = kebab-case description of what is tested

## describe + test naming

```ts
test.describe('Orders — Initial state and filters', () => {
  test('displays empty state when accessing the page @smoke @regression', ...);
  test('hides date fields when switching to Monthly mode @regression', ...);
});
```

Format: `[verb] <what> <condition>` — match the language used in the UI.

## AAA with test.step()

```ts
test('generates report when filters are filled', async ({ reportPage: page }) => {
  await test.step('Arrange — select category', async () => {
    await page.getByTestId('select-category').click();
    await page.locator('[role="option"]').first().click();
  });

  await test.step('Act — generate report', async () => {
    await page.getByTestId('btn-generate-report').click();
    await page.getByTestId('section-results').waitFor({ state: 'visible', timeout: 20_000 });
  });

  await test.step('Assert — panels visible', async () => {
    await expect(page.getByTestId('section-chart')).toBeVisible();
  });
});
```

Use `test.step()` in tests with 3+ interactions — improves CI report diagnostics.

## Tags

```ts
test('...', { tag: ['@smoke', '@regression'] }, async (...) => { ... });
```

| Tag | Meaning |
|-----|---------|
| `@smoke` | Critical path, runs on every deploy |
| `@regression` | Full regression suite |
| `@acl` | Access control / permission tests |

## Custom Fixtures — entry point

All specs import from `../../fixtures/custom/app-fixtures` (never from `@playwright/test` directly):

```ts
import { test, expect } from '../../fixtures/custom/app-fixtures';
```

`app-fixtures.ts` re-exports `expect` and all custom fixtures via a single `base.extend<>()` call.

## Decision: fixture vs beforeEach

- Same setup used in 3+ specs → Custom Fixture in `fixtures/custom/`
- Setup used in only 1 spec → `beforeEach` inline in the spec
