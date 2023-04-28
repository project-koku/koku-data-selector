
=============================
Azure: Customer-filtered data
=============================


Azure resource group and storage account
========================================

1. Create new resource group and storage account for filtered reports using these `instructions <https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal>`_

2. Setup read access for Cost Management
    a. In Azure Cloud Shell run the following to obtain the subscription ID: ``az account show --query "{subscription_id: id }"``
    b. Create Resource group role: ``az ad sp create-for-rbac -n "CostManagement" --role "Storage Account Contributor"  --scope /subscriptions/{subscriptionId}/resourceGroups/{resourceGroup1} --query '{"tenant": tenant, "client_id": appId, "secret": password}'``
    c. **Note** Keep a copy of your subscription_id, client_id, secret and tenant for creating your source in console.redhat.com

3. Create Cost Management source
    a. Follow sources wizard in console.redhat.com
    b. Be sure to create a **storage-only** Azure source type
    c. Visit https://console.redhat.com/api/cost-management/v1/sources/
    d. Find your source and note down its source_uuid, **required** for Azure function


4. Create billing export
    a. See section 1.3 `here <https://access.redhat.com/documentation/en-us/cost_management_service/2023/html/adding_a_microsoft_azure_source_to_cost_management/assembly-adding-azure-sources#configuring-an-azure-daily-export-schedule_adding-an-azure-source>`_ during creation select a storage account or create a new one.
    b. Make a note of the storage account in use for the cost export **Required** for function


5. Setup function to post reports
    a. Create Azure Time Trigger function using `VSCode <https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-configuration>`_
        i. Set the Time trigger function to run once a day

    b. Now you have a basic function created we need to add a few things
        i. Create a **requirements.txt** and add `these <https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/requirements.txt>`_
        ii. Replace the Trigger function with the following `code <https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/azure-function.txt>`_
        iii. **NOTE** Be sure to update the required vars
        iv. Deploy the function to Azure

    c. Setup blob access for function in Azure portal refer to `this <https://learn.microsoft.com/en-us/samples/azure-samples/functions-storage-managed-identity/using-managed-identity-between-azure-functions-and-azure-storage/>`_
        i. Navigate to Function App
        ii. Select identity in the blade
        iii. Turn on System assigned identity
        iv. Go to Azure role assignements
        v. Add the following roles for both storage accounts created previously  

        .. code-block::

            Storage Blob Data Contributor
            Storage Queue Data Contributor

    d. Optionally store credentials in vault: `Key vault Credentials`_
        i. Navigate to your function
        ii. Select Configuration under settings in the blade
        iii. Click New application setting
        iv. Name: *UsernameFromVault*
        v. Value: *@Microsoft.KeyVault(SecretUri=YOUR-USER-SECRET-URI)*
        vi. Save
        vii. Add Another application setting for: *PasswordFromVault*
        viii. Value: *@Microsoft.KeyVault(SecretUri=YOUR-PASSWORD-SECRET-URI)*
        ix. Make sure to Replace the URI's with your Secret URI's 

Key vault Credentials
=====================

1. From Azure key vaults - Create secret
2. Select the same resource group your function resides in
3. Set Key Vault Name
4. Within Access policy - Create new access policy
    i. Select Secret Management from templates
    ii. For principal find your Created function
    iii. Review and create policy

7. Review and create Vault
8. Click on new vault and select secrets in the blade
9. Generate/import two new secrets
    i. Secret names as follows *UsernameFromVault* and *PasswordFromVault*
    ii. Giving them a secret value matching your console.redhat.com username and password respectively

10. Click on each secret - select the version
11. Copy the secret Identifier URI
