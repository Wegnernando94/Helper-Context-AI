---
name: playwright-ci
description: CI/CD integration — Azure DevOps pipeline YAML, JUnit reporter, artifacts, env vars
---

# Playwright CI

## Azure DevOps pipeline (YAML)

```yaml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  BASE_URL: $(E2E_BASE_URL)
  E2E_EMAIL: $(E2E_EMAIL_SECRET)
  E2E_PASSWORD: $(E2E_PASSWORD_SECRET)

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'

  - script: npm ci
    displayName: 'Install dependencies'

  - script: npx playwright install --with-deps chromium
    displayName: 'Install Playwright browsers'

  - script: npx playwright test --reporter=junit,html,list
    displayName: 'Run E2E tests'
    continueOnError: true

  - task: PublishTestResults@2
    inputs:
      testResultsFormat: 'JUnit'
      testResultsFiles: 'results.xml'
      failTaskOnFailedTests: true

  - task: PublishPipelineArtifact@1
    condition: always()
    inputs:
      targetPath: 'playwright-report'
      artifact: 'playwright-report'
```

## Reporter config (playwright.config.ts)

```ts
reporter: [
  ['html'],
  ['junit', { outputFile: 'results.xml' }],
  ['list'],
],
```

## CI-specific rules

- `workers: 1` on CI — prevents race conditions on shared QA database
- `retries: 2` on CI — automatically retries flaky tests before marking failure
- `forbidOnly: !!process.env.CI` — prevents `test.only` from silently skipping other tests
- `BASE_URL` from pipeline variable — allows targeting different environments
- Screenshots and traces stored as pipeline artifacts for failure diagnosis
