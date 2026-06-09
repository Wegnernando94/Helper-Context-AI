---
name: playwright-advanced
description: Advanced patterns — multi-tab, clipboard, drag-and-drop, iframes, ACL tests, accessibility
---

# Playwright Advanced

## Multi-tab handling

```ts
const [newPage] = await Promise.all([
  page.context().waitForEvent('page'),
  page.getByTestId('btn-open-in-new-tab').click(),
]);
await newPage.waitForLoadState();
await expect(newPage.getByTestId('page-title')).toBeVisible();
```

## Clipboard

```ts
await page.evaluate(() => navigator.clipboard.writeText('test content'));
// Or read:
const text = await page.evaluate(() => navigator.clipboard.readText());
```

## Drag and drop

```ts
await page.getByTestId('draggable-item').dragTo(
  page.getByTestId('drop-target'),
);
```

## iframes

```ts
const iframe = page.frameLocator('[data-testid="embed-frame"]');
await iframe.getByTestId('btn-inside-frame').click();
```

## ACL tests (@acl tag)

ACL tests verify that unauthorized users cannot see or interact with protected elements:

```ts
test('viewer não deve ver botão de exclusão @acl', async ({ page }) => {
  // Load viewer storageState via project config
  await expect(page.getByTestId('btn-delete')).toBeHidden();
  await expect(page.getByTestId('btn-edit')).toBeHidden();
});
```

ACL specs live at the end of the feature's spec sequence (e.g., `07-acl.spec.ts`).

## Accessibility (WCAG 2.1 AA)

```ts
import { checkA11y } from 'axe-playwright';

test('página deve passar na auditoria de acessibilidade @a11y', async ({ page }) => {
  await checkA11y(page, undefined, {
    detailedReport: true,
    detailedReportOptions: { html: true },
  });
});
```

## Clock / datepicker

```ts
await page.clock.setFixedTime(new Date('2024-03-15'));
await page.goto('/reports/...');
// Date fields will show 2024-03-15 as "today"
```

Use `page.clock` instead of computing relative dates in assertions — makes tests deterministic.

## Flaky test prevention checklist

- [ ] No `waitForTimeout` anywhere
- [ ] All network waits use `waitForResponse` registered BEFORE the triggering action
- [ ] `workers: 1` on CI
- [ ] No shared mutable state between tests
- [ ] Fixtures clean up after themselves (teardown via `use()` pattern)
- [ ] `storageState` loaded at project level, not per-spec
