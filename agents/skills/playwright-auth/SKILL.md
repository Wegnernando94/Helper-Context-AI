---
name: playwright-auth
description: Authentication strategy — global storageState, SSO bypass, global.setup.ts pattern
---

# Playwright Auth

## Global auth setup (mandatory pattern)

```ts
// tests/global.setup.ts
import { test as setup } from '@playwright/test';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByTestId('input-email').fill(process.env.E2E_EMAIL!);
  await page.getByTestId('input-password').fill(process.env.E2E_PASSWORD!);
  await page.getByTestId('btn-login').click();
  await page.waitForURL('**/dashboard**');
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

## Rules

- `storageState` set at project level in `playwright.config.ts` — NEVER repeat `test.use({ storageState })` in individual specs
- `playwright/.auth/user.json` must be gitignored
- Tests that validate the login flow itself are the ONLY exception — they must NOT load storageState

## SSO / OAuth PKCE

Complete the full OAuth redirect flow once in `global.setup.ts` and save cookies + localStorage to `user.json`. Token refresh is handled transparently by storageState on subsequent runs.

## Multi-role auth

When tests require different permission levels, create separate auth files:

```ts
// playwright.config.ts projects
{ name: 'e2e-admin', use: { storageState: 'playwright/.auth/admin.json' } }
{ name: 'e2e-viewer', use: { storageState: 'playwright/.auth/viewer.json' } }
```
