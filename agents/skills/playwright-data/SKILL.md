---
name: playwright-data
description: Test data strategy — mock builders, fixtures/data/ folder, API-first setup, pure functions
---

# Playwright Data

## Folder conventions

```
tests/
  fixtures/
    custom/    # Custom Playwright fixtures (have Page dependency)
    data/      # Pure data builders (NO Page dependency)
```

Files in `fixtures/data/` must be pure TypeScript — no imports from `@playwright/test`.

## Naming convention for data files

```
tests/fixtures/data/<feature>.mock.ts
```

Example: `tests/fixtures/data/orders-report.mock.ts`

## Pure mock builder pattern

```ts
// fixtures/data/orders-report.mock.ts

export function buildCategoriesMock(): object {
  return {
    data: [
      { id: 1, name: 'Category Alpha' },
    ],
    meta: { total: 1, per_page: 15, current_page: 1, last_page: 1 },
  };
}

export function buildOrdersMock(typeId: number, parentId?: string | null): object {
  // ... domain-specific logic
}
```

Mock builders are imported by fixtures in `fixtures/custom/`, never directly by specs.

## API-first data setup

When a test needs existing data (order to cancel, supplier to edit), create it via API in the fixture — NOT by clicking through the UI.

```ts
// In fixture setup
const response = await request.post('/api/v1/orders', {
  data: buildRandomOrderData(),
});
const { id } = await response.json();
await use(id);

// Teardown
await request.delete(`/api/v1/orders/${id}`);
```

## Decision rule

- Is the UI interaction WHAT the test validates? → Use UI
- Is it just precondition state? → Use API

## Sensitive data — never hardcode

```ts
// .env.test (gitignored)
E2E_EMAIL=qa@example.com
E2E_PASSWORD=secret

// global.setup.ts
process.env.E2E_EMAIL!
process.env.E2E_PASSWORD!
```
