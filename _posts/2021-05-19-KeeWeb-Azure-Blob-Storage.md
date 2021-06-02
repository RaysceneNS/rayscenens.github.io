---
title: "KeeWeb Azure blob container storage"
tags: [KeeWeb, Azure, Blob Container]
---

Use azure blob storage to store KeePass files that can be shared among multiple users within your organization.

I created a plugin for KeeWeb that extends KeeWeb with the ability to load KeePass files stored within Azure Blob storage.

You can find this on [GitHub](https://github.com/RaysceneNS/keeweb)


To setup this plugin you will need to configure a few settings within the Azure Portal.


# Register your application on Azure Active Directory

1. Sign in the [Azure Portal](https://portal.azure.com/)
1. Select Azure Active Directory
1. Under Manage, select App Registrations > New registration
1. Enter a unique name for your application.
1. Under Supported Account Types choose 'Accounts in this organizational directory only (Default Directory only - Single tenant)
'
1. Under redirect URI, select Single-page application (SPA), enter the path to the web application that was setup earlier in the form https://YOUR_SERVER_HERE/oauth-result/azure.html
1. Select Register.
1. On the app overview page, note the Application (client) ID value, we will need this for later use.
1. On the app overview page, note the Directory (tenant) ID value, we will need this for later use.
1. Under Manage, select Authentication.
1. In the Implicit grant and hybrid flows section, select ID tokens. ID tokens are required because this app must sign in users and call an API.
1. Select Save.
1. Under Manage, select API permissions.
1. Select Add a permission, select Azure Storage, pick user_impersonation.
1. Click Grant admin consent for Default Directory.


# Grant access to the Storage Account 
1. Storage Account
1. Select Access Control (IAM)
1. Press Add > Role Assignment
1. Assign access to the users or groups that can access the files in this container, give each entry the Role 'Storage Blob Data Contributor' if you want them to have the ability to modify the data in the container.
1. Press Save.

# Configure CORS to enable the web application to make requests to the Storage Account API.
1. Open the Storage Account in Azure Portal
1. Select Settings > Resource sharing (CORS)
1. Add the URL for the web application, SELECT ALL for the allowed methods, enter * for the Allowed headers, enter * for the Exposed headers.
1. Press Save


# Enter configuration information in KeeWeb

Modify the file default-app-settings.js to include the following keys:

```js
    azure: true,
    azureClientId: 'your_client_id',
    azureTenantId: 'your_tenant_id',
    azureBlobContainer: 'https://your_blob_store.blob.core.windows.net'
```

azureClientId - Replace your_client_id with the Application (client) ID for the application you saved earlier.

azureTenantId - Replace your_tenant_id with the Directory (tenant) ID for the application you saved earlier.

azureBlobContainer - Set to the either the URL for the storage account root or optionally include the path to the container as well.
