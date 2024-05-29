# Run Playwright tests

## Run tests

```bash
npx playwright test
```

## Show test results

Open the generated report in the browser.

```bash
npx playwright show-report
```

## Run tests in headless mode

```bash
npx playwright test --headed
```

## Run tests in headless mode with a specific browser

```bash
npx playwright test --headed --project webkit
```

## Run tests with debug mode

```bash
npx playwright test --debug
```

## Generate a test with Playwright

```bash
npx playwright codegen https://tbernardo-echo-app-webapp-uat.azurewebsites.net/
```
