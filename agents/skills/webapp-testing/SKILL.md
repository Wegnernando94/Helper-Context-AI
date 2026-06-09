---
name: webapp-testing
description: Web application testing principles. E2E, Cypress, deep audit strategies.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Web App Testing

> Discover and test everything. Leave no route untested.

## 🔧 Runtime Scripts

**Execute these for automated browser testing:**

| Script                         | Purpose             | Usage                                                     |
| ------------------------------ | ------------------- | --------------------------------------------------------- |
| `scripts/playwright_runner.py` | Basic browser test  | `python scripts/playwright_runner.py https://example.com` |
|                                | With screenshot     | `python scripts/playwright_runner.py <url> --screenshot`  |
|                                | Accessibility check | `python scripts/playwright_runner.py <url> --a11y`        |

**Requires:** `pip install playwright && playwright install chromium`

---

## 1. Deep Audit Approach

### Discovery First

| Target        | How to Find                     |
| ------------- | ------------------------------- |
| Routes        | Scan app/, pages/, router files |
| API endpoints | Grep for HTTP methods           |
| Components    | Find component directories      |
| Features      | Read documentation              |

### Systematic Testing

1. **Map** - List all routes/APIs
2. **Scan** - Verify they respond
3. **Test** - Cover critical paths

---

## 2. Testing Pyramid for Web

```
        /\          E2E (Few)
       /  \         Critical user flows
      /----\
     /      \       Integration (Some)
    /--------\      API, data flow
   /          \
  /------------\    Component (Many)
                    Individual UI pieces
```

---

## 3. E2E Test Principles

### What to Test

| Priority | Tests                     |
| -------- | ------------------------- |
| 1        | Happy path user flows     |
| 2        | Authentication flows      |
| 3        | Critical business actions |
| 4        | Error handling            |

### E2E Best Practices

| Practice                     | Why                |
| ---------------------------- | ------------------ |
| Use data-testid              | Stable selectors   |
| Wait for elements            | Avoid flaky tests  |
| Clean state                  | Independent tests  |
| Avoid implementation details | Test user behavior |

---

## 4. Cypress Principles

### Core Concepts

| Concept | Use |
| --- | --- |
| `cy.intercept()` + `cy.wait('@alias')` | Wait for network calls before asserting |
| `cy.sessionLogin()` | Session-cached login — never per-test UI login |
| Single `beforeEach` | One consolidated hook per `describe` level |
| `Cypress.env()` | Access sensitive and environment variables |

### Configuration

| Setting     | Recommendation    |
| ----------- | ----------------- |
| Retries (runMode) | 2 on CI |
| Retries (openMode) | 0 locally |
| `testIsolation` | Always `true` — never `false` |
| Screenshots | on-failure        |
| Video       | retain-on-failure |

---

## 5. Visual Testing

### When to Use

| Scenario          | Value  |
| ----------------- | ------ |
| Design system     | High   |
| Marketing pages   | High   |
| Component library | Medium |
| Dynamic content   | Lower  |

### Strategy

- Baseline screenshots
- Compare on changes
- Review visual diffs
- Update intentional changes

---

## 6. API Testing Principles

### Coverage Areas

| Area           | Tests                       |
| -------------- | --------------------------- |
| Status codes   | 200, 400, 404, 500          |
| Response shape | Matches schema              |
| Error messages | User-friendly               |
| Edge cases     | Empty, large, special chars |

---

## 7. Test Organization

### File Structure

```
cypress/
├── e2e/                     # Specs organized by feature
│   ├── Login/
│   │   └── login.cy.js
│   ├── Dashboard/
│   │   └── dashboard.cy.js
│   └── Admin/
│       └── admin.cy.js
├── fixtures/                # Static JSON test data
├── reports/
│   └── validation/          # PASS/FAIL reports for Azure Test Plans
├── screenshots/
└── support/
    ├── tasks/
    │   └── index.js         # Node.js tasks (cy.task)
    ├── commands.js          # Custom commands (cy.sessionLogin, etc.)
    └── e2e.js               # Global config (viewport, login, interceptors)
cypress.config.js
cypress.env.example.json     # Public reference — always versioned
cypress.env.json             # NOT versioned — local sensitive variables
```

### Naming Convention

| Pattern       | Example                     |
| ------------- | --------------------------- |
| Feature-based | `login.cy.js`               |
| Descriptive   | `user-can-checkout.cy.js`   |

---

## 8. CI Integration

### Pipeline Steps

1. Install dependencies
2. Inject environment variables
3. Run tests (`cypress run`)
4. Upload artifacts (screenshots, reports)

### Execution Scripts

| Script | Viewport |
| --- | --- |
| `npm run cy:run:desktop` | 1366x768 |
| `npm run cy:run:tablet` | 1024x768 |
| `npm run cy:run:responsive` | desktop + tablet |
| `npm run cy:run:smoke` | grep `@smoke` |
| `npm run cy:run:regression` | full suite |

---

## 9. Anti-Patterns

| ❌ Don't            | ✅ Do          |
| ------------------- | -------------- |
| `cy.wait(ms)` hardcoded | `cy.wait('@alias')` after `cy.intercept()` |
| UI login on every test | `cy.sessionLogin()` in `beforeEach` |
| Multiple `beforeEach` same level | Single consolidated `beforeEach` |
| Hardcoded sensitive data | `Cypress.env('VAR')` with `{ log: false }` |
| CSS class selectors | `[data-testid="..."]` as primary selector |
| XPath | Absolutely forbidden |
| `testIsolation: false` | Always `true` |
| `if/else` on UI state | Separate test cases instead |

---

> **Remember:** E2E tests are expensive. Use them for critical paths only.
