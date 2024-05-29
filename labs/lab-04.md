# Lab 04 - Run Playwright Tests

## Description

This lab will demonstrate how to use Playwright to create and run tests in a software development project. The student will learn how to create a test script using Playwright and run it against a web application.

## Pre-requisites

- Install Node.js 18+. [More details here](https://nodejs.org/en/download/package-manager)

## Guide

### Step 01 - Init Playwright

Navigate to the `echo-webapp` folder on the terminal.

```bash
cd echo-webapp
```

Initialize the Playwright project.

```bash
npm init playwright@latest
```

You'll get several questions about the init, please select the following options:

- **Select a language**: TypeScript
- **Where to put your end-to-end tests?**: tests
- **Add a GitHub Actions workflow?**: No
- **Install Playwright browsers (can be done manually via 'npx playwright install')?**: Yes
- **playwright.config.ts already exists. Override it?**: No

### Step 02 - Configure Playwright

Edit the file `playwright.config.ts` on the folder `echo-webapp`.

Find the property `baseURL` and update it to the URL of the web application.

The property should look like this:

```typescript
baseURL: process.env.BASE_URL || '<URL_for_UAT_AppService>',
```

Then remove all files on the `tests` folder besides `echo.spec.ts`.

On folder `tests-examples` you can find a great example of Playwright usage for website <https://demo.playwright.dev/todomvc>. You can use as a good reference.

For this lab, you should delete that folder before proceed.

### Step 03 - Run Playwright Tests locally

First, take a look on file `echo.spec.ts` on the `tests` folder.

This file specifies all steps to be performed by Playwright to test the web application.

You can see 2 tests:

1) Access the home page and check if the title is correct.
2) Access the home page, enters a random text on textbox, clicks on `Make Echo` button and check if the result is correct.

Now, let's run the tests.

```bash
npx playwright test
```

### Step 04 - Show test results

Since all the tests are done and run successfully, let's see the results.

Open the generated report in the browser.

```bash
npx playwright show-report
```

Please take some time to navigate on this report and understand the results.

When you run the tests and you get any error, the report is automatically generated and you can see the error details.

When you click on the details of a test, you can get access the tracing of the test.
