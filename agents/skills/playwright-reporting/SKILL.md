---
name: playwright-reporting
description: Validation reports — PASS/FAIL format, test.step() diagnostics, artifact structure
---

# Playwright Reporting

## Validation report format

After running tests, produce a report in this structure:

```markdown
## Validation Report — <Feature Name>

**Date:** YYYY-MM-DD  
**Environment:** https://app.example.com  
**Browser:** Chromium  

### Results

| Test | Status | Observation |
|------|--------|-------------|
| ORD-01 — displays empty state | ✅ PASS | — |
| ORD-01 — switches to Monthly mode | ✅ PASS | — |
| ORD-02 — validates required filter | ❌ FAIL | `select-category` not found |

### Failures

**ORD-02 — validates required filter**
- Error: `getByTestId('select-category')` — no element found
- Trace: playwright-report/trace-ord02.zip
- Screenshot: playwright-report/screenshots/ord02-failure.png
```

## test.step() in reports

`test.step()` labels appear in the HTML report and JUnit XML, making it possible to pinpoint which step of a multi-action test failed. Always use `test.step()` in tests with 3+ interactions.

## Artifacts

| Artifact | When generated | Purpose |
|----------|---------------|---------|
| `playwright-report/` | Always | HTML report with full timeline |
| `results.xml` | Always | JUnit for CI Test Plans |
| `playwright-report/trace-*.zip` | On retry | Step-by-step trace replay |
| `playwright-report/screenshots/` | On failure | Visual failure evidence |

## CI Test Plans integration

JUnit `results.xml` is consumed by the `PublishTestResults` task, which:
- Creates/updates test run in the CI platform
- Maps test cases by name to existing Test Plan items
- Marks suite as passed/failed automatically
