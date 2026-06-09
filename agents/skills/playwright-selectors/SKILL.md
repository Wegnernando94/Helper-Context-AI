---
name: playwright-selectors
description: Selector strategy — 3-tier fallback, data-testid as primary, DevExtreme patterns
---

# Playwright Selectors

## 3-tier fallback hierarchy

| Priority | Selector | When to use |
|----------|----------|-------------|
| 1st | `getByTestId('...')` | Always — preferred when `data-testid` exists |
| 2nd | `getByPlaceholder('...')` | Input fields without `data-testid` |
| 3rd | `getByRole('option', { name })` | Native browser elements (dropdowns, options) |

Never use CSS selectors, XPath, or class-based selectors.

## DevExtreme patterns

DevExtreme components render complex DOM. Use these patterns:

```ts
// SelectBox — click to open, then pick option by role
await page.getByTestId('select-period-type').click();
await page.getByRole('option', { name: 'Mensal' }).click();

// First remote option (when content is dynamic)
await page.getByTestId('select-category').click();
await page.locator('[role="option"]').first().click();

// DateBox — interact via the input inside
await page.getByTestId('input-start-date').fill('2024-01-01');

// TreeList expand node
await page.getByTestId('btn-expand-node').click();
```

## Hidden/visible assertions

```ts
await expect(page.getByTestId('section-chart')).toBeHidden();
await expect(page.getByTestId('section-chart')).toBeVisible();
```

Prefer `toBeHidden()` over `not.toBeVisible()` — same behavior, cleaner intent.

## What NOT to do

```ts
// WRONG — brittle class selectors
page.locator('.dx-selectbox-container input')

// WRONG — index-based
page.locator('button').nth(2)

// WRONG — text content that changes with i18n
page.locator('text=Save')
```
