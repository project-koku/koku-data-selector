
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
    b. Make a note of the storage account, container name, export dir and export name **Required** for function


5. Setup function to post reports
    a. Create Azure Time Trigger function using `VSCode <https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-configuration>`_
        i. Set the Time trigger function to run once a day

    b. Now you have a basic function created we need to add a few things
        i. Create a **requirements.txt** and add `these <https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/requirements.txt>`_
        ii. Add `Function Code and Queries`_,
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
        i. See `Adding Vault Creds To functions`_

Collecting Finalized Data
=========================

1. Setup function to send finalized billing reports
    a. Create Azure Time Trigger function using `VSCode <https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-configuration>`_
        i. Set the Time trigger function to run once a month after the 3rd of the month (after Azure finialises billing data)
        ii. Uncomment the following lines 

        ..code-block::

            # month_end = now.replace(day=1) - timedelta(days=1)
            # month_start = last_month_end.replace(day=1)
            # month = last_month.strftime("%m")
            # day = last_month.strftime("%d")


    b. Now you have a basic function created we need to add a few things
        i. Create a **requirements.txt** and add `these <https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/requirements.txt>`_
        ii. Add `Function Code and Queries`_,
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
        i. See `Adding Vault Creds To functions`_

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

Adding Vault Creds To functions
===============================
1. Navigate to your function
2. Select Configuration under settings in the blade
3. Click New application setting
4. Name: *UsernameFromVault*
5. Value: *@Microsoft.KeyVault(SecretUri=YOUR-USER-SECRET-URI)*
6. Save
7. Add Another application setting for: *PasswordFromVault*
8. Value: *@Microsoft.KeyVault(SecretUri=YOUR-PASSWORD-SECRET-URI)*
9. Make sure to Replace the URI's with your Secret URI's 

Function Code and Queries
=========================
* For standard Hybrid Commited Spend queries use the default `azure_function <https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/azure-function.txt>`_
* For custom queries non HCS we need to edit line 53 in the above function code.
    * Initial query to grab all data: **filtered_data = df**
    * To filter the data you need to add some dataframe filtering, see Examples:
        * Exact matching: **df.loc[(df["publisherType"] == "Marketplace")]** would filter out all data that does not have a publisherType of Marketplace.
        * Contains: **df.loc[df["publisherName"].astype(str).str.contains("Red Hat")]** would filter all data that does not contain Red Hat in the publisherName.
    * It's also possible to stack these by using **&** (for AND) and **|** (for OR) with your **df.loc** clause.
    * Examples:
        1. **subscriptionId** Used to filter specific subscriptions.
        2. **resourceGroup** Used to filter specific resource groups.
        3. **resourceLocation** Used to filter data in a specifc region.
        4. **resourceType**, **instanceId** Used to filter resource types or by a specifc instance.
        5. **serviceName**, **serviceTier**, **meterCategory** and **meterSubcategory** can be used to filter specifc service types.
    * Once your custom query is built just replace line 53 with your revised version.
