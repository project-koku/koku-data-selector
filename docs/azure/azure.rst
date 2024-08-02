
=============================
Azure: Customer-filtered data
=============================


Azure resource group and storage account
========================================
*Prerequisites:*
    - A console.redhat.com service account is required
    - The service account must have the correct roles assigned in C.R.C for Cost management access
    - Filtered CSV files *MUST* have the required columns as defined in `Azure filtering required columns`_

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
            # month_start = month_end.replace(day=1)
            # year = month_start.strftime("%Y")
            # month = month_start.strftime("%m")
            # day = month_start.strftime("%d")


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

1. Go to *Key Vaults* in Azure
2. Click Create, to create new secret
3. Select the same resource group your function resides in
4. Add Key Vault Name - Next
5. Under *Permissions model* - Choose Vault access policy
6. Within *Access policies* - Create new policy
    i. Select Secret Management from templates
    ii. For principal find your Created function
    iii. Review and create policy

7. Review and create Vault
8. Click on new vault and select secrets in the blade
9. Generate/import two new secrets
    i. Secret names as follows *ClientIdFromVault* and *ClientSecretFromVault*
    ii. Giving them a secret value matching your console.redhat.com service account client_id and client_secret respectively

10. Click on each secret - select the version
11. Copy the secret Identifier URI

Adding Vault Creds To functions
===============================
1. Navigate to your function
2. Select Environment variables under settings in the blade
3. Click add to create variable
4. Name: *ClientIdFromVault*
5. Value: *@Microsoft.KeyVault(SecretUri=YOUR-CLIENT-ID-URI)*
6. Save
7. Add Another application setting for: *ClientSecretFromVault*
8. Value: *@Microsoft.KeyVault(SecretUri=YOUR-CLIENT-SECRET-URI)*
9. Make sure to Replace the URI's with your Secret URI's 

Function Code and Queries
=========================
* The default script has the option of Hybrid commited spend or RHEL subscription filtering `azure_function <https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/azure-function.txt>`_
    * All you need to do is uncomment the relevant line, either `filtered_data = hcs_filtering(df)` OR `filtered_data = rhel_filtering(df)` NOT both
* For custom queries you will need to write your own filtering.
    * Initial query to grab all data: **filtered_data = df**
    * To filter the data you need to add some dataframe filtering, see Examples:
        * Exact matching: **df.loc[(df["publishertype"] == "Marketplace")]** would filter out all data that does not have a publisherType of Marketplace.
        * Contains: **df.loc[df["publishername"].astype(str).str.contains("Red Hat")]** would filter all data that does not contain Red Hat in the publisherName.
    * It's also possible to stack these by using **&** (for AND) and **|** (for OR) with your **df.loc** clause.
    * Examples:
        1. **subscriptionid** Used to filter specific subscriptions.
        2. **resourcegroup** Used to filter specific resource groups.
        3. **resourcelocation** Used to filter data in a specifc region.
        4. **resourcetype** Used to filter resource types.
        5. **servicename**, **servicetier**, **metercategory** and **metersubcategory** can be used to filter specifc service types.
    * Once your custom query is built just replace line 53 with your revised version.

Azure filtering required columns
================================

* Below is a list of columns that *MUST* be included and populated for Cost management to process the report correctly
* These columns *MUST* not include spaces, dashes, underscores and *MUST* be in lower case

    ..code block:

    'accountname',
    'accountownerid',
    'additionalinfo',
    'availabilityzone',
    'billingaccountid',
    'billingaccountname',
    'billingcurrency',
    'billingperiodenddate',
    'billingperiodstartdate',
    'billingprofileid',
    'billingprofilename',
    'chargetype',
    'consumedservice',
    'costcenter',
    'costinbillingcurrency',
    'date',
    'effectiveprice',
    'frequency',
    'invoicesectionid',
    'invoicesectionname',
    'isazurecrediteligible',
    'metercategory',
    'meterid',
    'metername',
    'meterregion',
    'metersubcategory',
    'offerid',
    'partnumber',
    'paygprice',
    'planname',
    'pricingmodel',
    'productname',
    'productorderid',
    'productordername',
    'publishername',
    'publishertype',
    'quantity',
    'reservationid',
    'reservationname',
    'resourcegroup',
    'resourceid',
    'resourcelocation',
    'resourcename',
    'resourcerate',
    'resourcetype',
    'servicefamily',
    'serviceinfo1',
    'serviceinfo2',
    'servicename',
    'servicetier',
    'subscriptionid',
    'subscriptionname',
    'tags',
    'term',
    'unitofmeasure',
    'unitprice'

* Some of these required columns differ depending on the base report type in use, the example script handles these differences already with the following:

    ..code block:

    column_translation = {"billingcurrencycode": "billingcurrency", "currency": "billingcurrency", "instanceid": "resourceid", "instancename": "resourceid", "pretaxcost": "costinbillingcurrency", "product": "productname", "resourcegroupname": "resourcegroup", "subscriptionguid": "subscriptionid", "usage_quantity": "quantity"}
