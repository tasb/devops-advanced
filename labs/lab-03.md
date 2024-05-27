# Lab 03 - Using Feature Flags

## Table of Contents

- [Description](#description)
- [Prerequisites](#prerequisites)
- [Guide](#guide)
  - [Step 01 - Create an App Configuration Service](#step-01---create-an-app-configuration-service)
  - [Step 02 - Configure Azure Permissions](#step-02---configure-azure-permissions)
  - [Step 03 - Use the Feature Flag](#step-03---use-the-feature-flag)
  - [Step 04 - Deploy the Changes](#step-04---deploy-the-changes)
  - [Step 05 - Disable the Feature Flag](#step-05---disable-the-feature-flag)
- [Conclusion](#conclusion)

## Description

This lab will demonstrate how to use feature flags in a software development project. The student will learn how to use the App Configuration service in Azure to create and manage feature flags.

## Prerequisites

- Lab 01 completed

## Guide

### Step 01 - Create an App Configuration Service

Let's add the Azure App Configuration and the feature flag on Terraform code.

Edit the file `main.tf` on the folder `deploy/terraform/echo-webapp`.

Find where you have the creation of `azurerm_service_plan` and add the following code **before** that resource:

```hcl
resource "azurerm_app_configuration" "appconfig" {
  name                = "${var.appName}-${var.appServiceName}-${var.env}-config"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "standard"
}

resource "azurerm_app_configuration_feature" "getlogsff" {
  configuration_store_id = azurerm_app_configuration.appconfig.id
  description            = "GetLogs button"
  name                   = "GetLogs"
  enabled                = true
}
```

Now let's edit the App Service code to automatically include the connection string to the App Configuration.

Find the resource `azurerm_app_service` and add update to have this final code:

```hcl
resource "azurerm_linux_web_app" "webapp" {
  name                = "${var.appName}-${var.appServiceName}-${var.env}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.appserviceplan.id

  app_settings = {
    "AppConfigConnString" = azurerm_app_configuration.appconfig.primary_read_key[0].connection_string
  }

  site_config {
    always_on = true
    application_stack {
      dotnet_version = "6.0"
    }
  }
}
```

To validate, navigate to Azure Portal and verify:

- The App Configuration service was created
- The feature flag was created
- The App Service has the connection string to the App Configuration

## Step 02 - Configure Azure Permissions

To allow the Terraform to create the feature flag, we need to give permissions to the App Configuration service.

Open the Azure Portal and navigate to resource group `<your-prefix>-echo-app-webapp-uat-rg`.

Please, replace `<your-prefix>` with your prefix.

Now click on `Access control (IAM)` and then click on `Add role assignment`.

Search and select the role `App Configuration Data Owner` and click `Next`.

On `Members`, click on `+ Select members` and search for the service principal you created on the first lab.

The SP name has the format `<PREFIX>-github-workflow`.

Now, click on `Review + assign` and then click on `Assign`.

Do the same process for the resource group `<your-prefix>-echo-app-webapp-prod-rg`.

## Step 03 - Use the Feature Flag

Now let's use the feature flag in the code.

Let's add the reference to needed packages in the `echo-webapp/echo-webapp.csproj` file:

```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.Azure.AppConfiguration.AspNetCore" Version="7.2.0" />
    <PackageReference Include="Microsoft.FeatureManagement.AspNetCore" Version="3.3.1" />
  </ItemGroup>
```

Paste the code before the `</Project>` tag.

Now let's add the feature flag in the `echo-webapp/Program.cs` file.

Find the line with this content `var app = builder.Build();` and add the following code before that line:

```csharp
// Add Azure App Configuration middleware to the container of services.
builder.Services.AddAzureAppConfiguration();
// Add feature management to the container of services.
builder.Services.AddFeatureManagement();

// Load configuration from Azure App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(builder.Configuration["AppConfigConnString"]);
    // Load all feature flags with no label
    options.UseFeatureFlags();
});
```

Now let's use the feature flag on the web application.

Edit the file `echo-webapp/Pages/_ViewImports.cshtml` and add the following code:

```csharp
@addTagHelper *, Microsoft.FeatureManagement.AspNetCore
```

Finally, let's use the feature flag in the `echo-webapp/Pages/Index.cshtml` file.

Let's include the following code on line 22:

```html
    <feature name="GetLogs">
        <div class="display-4">
            <button type="submit" class="btn btn-primary">Get Logs</button>
        </div>
    </feature>
```

At the end this file should have the following content:

```html
@page
@model IndexModel

@{
    ViewData["Title"] = "Home page";
    ViewData["Hostname"] = System.Net.Dns.GetHostName();
}

<div class="text-center">
    <h1 class="display-4" id="mainTitle">Echo WebApp</h1>
    <form class="form-group" method="post">
    <div class="form-group">
        <div class="display-4">
            <input type="text" class="form-control form-control-lg" name="Message">
        </div>
    </div>
    <div class="form-group">
        <div class="display-4">
            <button type="submit" class="btn btn-primary">Make Echo!</button>
        </div>
    </div>
    <feature name="GetLogs">
        <div class="display-4">
            <button type="submit" class="btn btn-primary">Get Logs</button>
        </div>
    </feature>
    
    <h2 id="echoResult">@ViewData["EchoedMessage"]</h2>
</form>
</div>
```

### Step 04 - Deploy the Changes

Now let's deploy the changes to Azure.

To do that proceed to implement a full pull request cycle as you've done on previous labs.

After the pull request is merged, the GitHub Actions will deploy the changes to Azure.

Navigate to the App Service URL and verify that the feature flag is working, you should see a button `Get Logs` on the page.

### Step 05 - Disable the Feature Flag

Now let's disable the feature flag.

Navigate to the Azure Portal and open the App Configuration service, then click on the option `Feature Manager`.

Check the feature flag `GetLogs` and click on the toggle to disable the feature flag.

Wait for about 1 minute and refresh the page on the App Service URL.

Now you should not see the button `Get Logs` on the page.

## Conclusion

In this lab, you learned how to use feature flags in a software development project. You learned how to create and manage feature flags using the App Configuration service in Azure.
