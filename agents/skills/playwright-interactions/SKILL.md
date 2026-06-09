---
name: playwright-interactions
description: Interaction patterns — file upload, network interception, waitForResponse, DevExtreme specifics
---

# Playwright Interactions

## Zero explicit waits — ABSOLUTE PROHIBITION

```ts
// FORBIDDEN
await page.waitForTimeout(2000);

// CORRECT — wait for specific condition
await page.getByTestId('section-result').waitFor({ state: 'visible', timeout: 20_000 });
await page.waitForURL('**/dashboard**');
await page.waitForResponse(resp => resp.url().includes('/api/orders'));
```

## Network interception (page.route)

Register routes BEFORE `page.goto()`:

```ts
// In fixture, before navigation
await page.route('**/api/v1/orders**', async (route) => {
  const url = new URL(route.request().url());
  const typeId = parseInt(url.searchParams.get('type_id') ?? '1', 10);
  return route.fulfill({ json: buildOrdersMock(typeId) });
});

await page.goto('/orders/report', { waitUntil: 'commit' });
```

## waitForResponse pattern

```ts
// Register the wait BEFORE the action that triggers the request
const refresh = page.waitForResponse(
  resp => resp.url().includes('/api/orders') && resp.request().method() === 'GET',
  { timeout: 45_000 },
);
await page.getByTestId('btn-apply-filter').click();
await refresh; // now wait for the response
```

## File upload (setInputFiles)

```ts
await page.locator('[data-testid="input-file"]').setInputFiles({
  name: 'document.pdf',
  mimeType: 'application/pdf',
  buffer: Buffer.from(fileContent, 'utf8'),
});
```

Never use OS file dialog — always `setInputFiles` with buffer.

## DevExtreme SelectBox — remote options

```ts
// Known option name
await page.getByTestId('select-period-type').click();
await page.getByRole('option', { name: 'Monthly' }).click();

// First available option (dynamic content)
await page.getByTestId('select-category').click();
const option = page.locator('[role="option"]').first();
await option.waitFor({ state: 'visible', timeout: 15_000 });
await option.click();
```

## Visibility assertions after interactions

```ts
// After switching a toggle
await expect(page.getByTestId('input-start-date')).toBeHidden();
await expect(page.getByTestId('select-year')).toBeVisible();
```

Always assert both what appears AND what disappears after an interaction.
