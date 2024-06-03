# Lab 04 - Load and Automated Test

## Table of Contents

- [Description](#description)
- [Guide](#guide)
  - [Step 01 - Add Playwright to the GitHub Actions workflow](#step-01---add-playwright-to-the-github-actions-workflow)
  - [Step 02 - Run Playwright tests](#step-02---run-playwright-tests)
  - [Step 03 - Create JMETER scripts](#step-03---create-jmeter-scripts)
  - [Step 04 - Prepare GitHub Environments](#step-04---prepare-github-environments)
  - [Step 05 - Add destroy stage](#step-05---add-destroy-stage)
  - [Step 06 - Create Azure Loading Test resource](#step-06---create-azure-loading-test-resource)
  - [Step 07 - Let's automate our tests](#step-07---let's-automate-our-tests)
  - [Step 08 - Destroy the UAT environment](#step-08---destroy-the-uat-environment)
- [Conclusion](#conclusion)

## Description

This lab will demonstrate how to use JMeter to create and run load tests in a software development project. The student will learn how to create a test script using JMeter and run it against a web application.

## Guide

### Step 01 - Add Playwright to the GitHub Actions workflow

Let's create a new branch for this lab.

First step is always to get last version of the main branch.

```bash
git checkout main
git pull
```

Now, let's create a new branch.

```bash
git checkout -b lab-05/playwright
```

Now, let's add Playwright to the GitHub Actions workflow.

Edit the file `.github/workflows/echo-webapp.yml` and add the following content on line #135 (after `logout` action):

```yaml
- uses: actions/checkout@v4
- uses: actions/setup-node@v4
  with:
    node-version: 18
- name: Install dependencies
  run: npm ci
  working-directory: echo-webapp
- name: Install Playwright Browsers
  run: npx playwright install --with-deps
  working-directory: echo-webapp
- name: Run Playwright tests
  run: npx playwright test
  working-directory: echo-webapp
- uses: actions/upload-artifact@v4
  if: ${{ !cancelled() }}
  with:
    name: playwright-report
    path: echo-webapp/playwright-report/
    retention-days: 30
```

Pay attention to the indentation of the file, these action should be at the same level of the `logout` action.

Now let's push the changes to the repository.

```bash
git commit -a -m "Add Playwright to the GitHub Actions workflow"
git push -u origin lab-05/playwright
```

### Step 02 - Run Playwright tests

Now, let's run the Playwright tests.

Navigate to GitHub and create a pull request from the branch `lab-05/playwright` to the main branch.

After the pull request is created, you should see the GitHub Actions checks run.

After the checks are done, you can complete the pull request.

After completing the Pull Request, navigate to the Actions tab and check the full CI/CD pipeline is running.

When UAT stage is done, you can scroll down and see the Artifacts section.

Click on the `playwright-report` artifact, unzip the downloaded file and open the `index.html` file.

On that webpage you can have access to the full report created by Playwright.

### Step 03 - Create JMETER scripts

Now, let's create a JMeter script to run load tests on the web application.

First, let's create a new branch for this task

```bash
git checkout main
git pull
git checkout -b lab-05/jmeter
```

Let's create a new folder called `jmeter` on `echo-webapp` folder.

```bash
mkdir echo-webapp/jmeter
```

Now, let's create a new file called `echo-webapp.jmx` on the `jmeter` folder with the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.4.1">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Test Plan" enabled="true">
      <stringProp name="TestPlan.comments"></stringProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.tearDown_on_shutdown">true</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments"/>
      </elementProp>
      <stringProp name="TestPlan.user_define_classpath"></stringProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Thread Group" enabled="true">
        <stringProp name="ThreadGroup.on_sample_error">continue</stringProp>
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController">
          <boolProp name="LoopController.continue_forever">false</boolProp>
          <stringProp name="LoopController.loops">1</stringProp>
        </elementProp>
        <stringProp name="ThreadGroup.num_threads">10</stringProp>
        <stringProp name="ThreadGroup.ramp_time">1</stringProp>
        <longProp name="ThreadGroup.start_time">1634713200000</longProp>
        <longProp name="ThreadGroup.end_time">1634713200000</longProp>
        <boolProp name="ThreadGroup.scheduler">false</boolProp>
        <stringProp name="ThreadGroup.duration"></stringProp>
        <stringProp name="ThreadGroup.delay"></stringProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="HTTP Request" enabled="true">
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
            <collectionProp name="Arguments.arguments"/>
          </elementProp>
          <stringProp name="HTTPSampler.domain">{your-prefix}-echo-app-webapp-uat.azurewebsites.net</stringProp>
          <stringProp name="HTTPSampler.port"></stringProp>
          <stringProp name="HTTPSampler.protocol">https</stringProp>
          <stringProp name="HTTPSampler.contentEncoding"></stringProp>
          <stringProp name="HTTPSampler.path">/</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
          <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
          <boolProp name="HTTPSampler.auto_redirects">false</boolProp>
          <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
          <boolProp name="HTTPSampler.DO_MULTIPART_POST">false</boolProp>
          <stringProp name="HTTPSampler.embedded_url_re"></stringProp>
          <stringProp name="HTTPSampler.connect_timeout"></stringProp>
          <stringProp name="HTTPSampler.response_timeout"></stringProp>
        </HTTPSamplerProxy>
        <hashTree/>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

Notice that you need to replace the `{your-prefix}` with your Azure prefix.

Let's create a new folder called `jmeter` on `echo-api` folder.

```bash
mkdir echo-api/jmeter
```

Now, let's create a new file called `echo-api.jmx` on the `jmeter` folder with the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.4.1">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Test Plan" enabled="true">
      <stringProp name="TestPlan.comments"></stringProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.tearDown_on_shutdown">true</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments"/>
      </elementProp>
      <stringProp name="TestPlan.user_define_classpath"></stringProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Thread Group" enabled="true">
        <stringProp name="ThreadGroup.on_sample_error">continue</stringProp>
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController">
          <boolProp name="LoopController.continue_forever">false</boolProp>
          <stringProp name="LoopController.loops">1</stringProp>
        </elementProp>
        <stringProp name="ThreadGroup.num_threads">10</stringProp>
        <stringProp name="ThreadGroup.ramp_time">1</stringProp>
        <longProp name="ThreadGroup.start_time">1634713200000</longProp>
        <longProp name="ThreadGroup.end_time">1634713200000</longProp>
        <boolProp name="ThreadGroup.scheduler">false</boolProp>
        <stringProp name="ThreadGroup.duration"></stringProp>
        <stringProp name="ThreadGroup.delay"></stringProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="HTTP Request" enabled="true">
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
            <collectionProp name="Arguments.arguments"/>
          </elementProp>
          <stringProp name="HTTPSampler.domain">{your-prefix}-echo-app-api-uat.azurewebsites.net</stringProp>
          <stringProp name="HTTPSampler.port"></stringProp>
          <stringProp name="HTTPSampler.protocol">https</stringProp>
          <stringProp name="HTTPSampler.contentEncoding"></stringProp>
          <stringProp name="HTTPSampler.path">/echo/jmeter</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
          <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
          <boolProp name="HTTPSampler.auto_redirects">false</boolProp>
          <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
          <boolProp name="HTTPSampler.DO_MULTIPART_POST">false</boolProp>
          <stringProp name="HTTPSampler.embedded_url_re"></stringProp>
          <stringProp name="HTTPSampler.connect_timeout"></stringProp>
          <stringProp name="HTTPSampler.response_timeout"></stringProp>
        </HTTPSamplerProxy>
        <hashTree/>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

Notice that you need to replace the `{your-prefix}` with your Azure prefix.

If you want, you can download JMeter from [here](https://jmeter.apache.org/download_jmeter.cgi) and run the tests locally.

This tool needs Java to run, so make sure you have Java installed on your machine.

### Step 04 - Prepare GitHub Environments

Now, let's prepare the GitHub environments to run the JMeter tests.

Since we want to make UAT a disposable environment, we will add a reviewer to the UAT environment to allow the test to run.

Navigate to the repository settings and click on the `Environments` tab.

Click on the `UAT` environment and add yourself as a reviewer, like you did on `Prod` environment.

Don't forget to click on `Save protection rules` after adding the reviewer.

### Step 05 - Add destroy stage

Now, let's add a new stage to the GitHub Actions workflow to destroy the UAT environment after the tests are done.

Edit the file `.github/workflows/echo-webapp.yml` and add the following content after the previously added lines to run Playwright tests:

```yaml
destroy-uat:
  if: github.event_name != 'pull_request'
  environment: 
    name: uat
  runs-on: ubuntu-latest
  needs: uat

  env:
    ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
    ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
    ARM_USE_OIDC: true

  steps:
  - uses: actions/download-artifact@v3
    with:
      name: ${{ env.ARTIFACT_NAME }}-iac
      path: ./terraform
   
  - uses: azure/login@v1
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  - uses: hashicorp/setup-terraform@v2
    with:
      terraform_wrapper: false

  - name: terraform init
    run: |
      cd ./terraform
      export
      terraform init -backend-config="key=echoapp.webapp.uat.tfstate"

  - name: terraform destroy
    run: |
      cd ./terraform
      terraform destroy -var="env=uat" -auto-approve

  - name: logout
    run: |
      az logout
```

Please pay attention to the indentation of the file, this stage should have the same indentation as the `uat` stage.

Now edit the file `.github/workflows/echo-api.yml` and add the following content before `prod` stage:

```yaml
destroy-uat:
  if: github.event_name != 'pull_request'
  environment: 
    name: uat
  runs-on: ubuntu-latest
  needs: uat

  env:
    ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
    ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
    ARM_USE_OIDC: true

  steps:
  - uses: actions/download-artifact@v3
    with:
      name: ${{ env.ARTIFACT_NAME }}-iac
      path: ./terraform
    
  - uses: azure/login@v1
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  - uses: hashicorp/setup-terraform@v2
    with:
      terraform_wrapper: false

  - name: terraform init
    run: |
      cd ./terraform
      export
      terraform init -backend-config="key=echoapp.api.uat.tfstate"

  - name: terraform destroy
    run: |
      cd ./terraform
      terraform destroy -var="dbPassword=${{ secrets.DB_PASSWORD }}" -var="env=uat" -auto-approve

  - name: logout
    run: |
      az logout
```

Please pay attention to the indentation of the file, this stage should have the same indentation as the `uat` stage.

### Step 06 - Create Azure Loading Test resource

This resource is not yet fully compatible with Terraform. Because of that you need to create it manually.

Navigate to Azure Portal and start to create a new resource group.

To create a new resource group, navigate to [Create a resource](https://portal.azure.com/#create/Microsoft.ResourceGroup) and fill the form with the following information:

- **Subscription**: Select your subscription
- **Resource group**: <your-prefix>-echo-app-loadtest-uat-rg
- **Region**: West Europe

Remember to replace `<your-prefix>` with your Azure prefix.

After creating the resource group, search for `Load Testing` on the search bar and click on the `Azure Load Testing` service.

Now, click on the `+ Create` button and fill the form with the following information:

- **Subscription**: Select your subscription
- **Resource group**: <your-prefix>-echo-app-loadtest-uat-rg
- **Name**: <your-prefix>-echo-app-uat-loadtest
- **Region**: West Europe

Now, click on the `Review + create` button and then on the `Create` button.

After the resource is created, click on the `Go to resource` button.

Then, click on `Tests` on the left menu and then on the `+ Create` button.

On the dropdown, select `Upload a script`.

Create a new test with the following information:

- **Name**: EchoWebappTest
- **Run test after creation**: Unchecked

Click on `Next` button.

On `Test Plan` tab, on `Choose files` option find the `echo-webapp.jmx` file you created before.

Then click on the `Upload` button.

You'll see the test being uploaded and a new line appears on the table.

Navigate to `Load` tab and fill the form with the following information:

- **Engine instances**: 10

Then, click on `+ Add/Edit Regions` button and select the region `North Europe`.

You can check that your engines will be split 50%/50% between the regions.

Finally, navigate to `Monitoring` tab.

Click on `+ Add/Modify` and on the list you get select all your resources related with UAT environment: `<your-prefix>-echo-app-api-uat` (App Service), `<your-prefix>-echo-app-webapp-uat` (App Service), `<your-prefix>-echodb-uat-psql` (PSQL Database).

Now click on `Review + create` button and then on the `Create` button.

Finally, repeat the same steps to create a new test for the `echo-api.jmx` file.

For that test use the following information:

- **Name**: EchoApiTest
- **Run test after creation**: Unchecked

For next properties, use the same values as the `EchoWebappTest`.

### Step 07 - Let's automate our tests

Now you have everything set up to run your tests.

Let's push the changes to the repository.

```bash
git add -A
git commit -m "Add JMeter scripts and destroy stage"
git push -u origin lab-05/jmeter
```

Now, navigate to GitHub and create a pull request from the branch `lab-05/jmeter` to the main branch.

After the pull request is created, you should see the GitHub Actions checks run.

After the checks are done, you can complete the pull request.

After completing the Pull Request, navigate to the Actions tab and check the full CI/CD pipeline is running.

Click on one of the pipelines and check the full pipeline running.

You can observe that on the flow diagram you have a new stage called `destroy-uat` that will destroy the UAT environment after the tests are done.

And you can observe too that you have two arrows pointing out from the `uat` stage, one to the `destroy-uat` stage and another to the `prod` stage.

Due to the change on the `UAT` environment, you need to approve the deployment to UAT.

After the `uat` stage is done, navigate to Azure Portal and check the `Azure Load Testing` service.

Click on `Tests` on the left menu and you should see the tests you created.

Click on one of the tests and then click on the `Run` button.

Do the same for the other test.

Now wait some time and check the results of the tests. Ypu can navigate on the report to see all metrics you get from this type of tests.

### Step 08 - Destroy the UAT environment

After the tests are done, go back to the GitHub Actions and approve the `destroy-uat` stage.

After the stage is done, navigate to Azure Portal and check the resource groups related with UAT environment were deleted.

Do the same approval to the other GitHub Actions pipeline.

## Conclusion

In this lab, you learned how to create and run load tests using JMeter and how to automate the process using GitHub Actions.
