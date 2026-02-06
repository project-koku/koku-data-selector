# Azure: Customer-filtered data

## Azure resource group and storage account

**Prerequisites:**
- A console.redhat.com service account is required.
- The service account must have at least the `settings.write` permission in the Red Hat Hybrid Console for Cost management access.
- Filtered CSV files *MUST* have the required columns as defined in [Azure filtering required columns](#azure-filtering-required-columns)

1. Create new resource group and storage account for filtered reports using these [instructions](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal)

2. Setup read access for Cost Management
    1. In Azure Cloud Shell run the following to obtain the subscription ID: `az account show --query "{subscription_id: id }"`
    2. Create Resource group role: `az ad sp create-for-rbac -n "CostManagement" --role "Storage Account Contributor"  --scope /subscriptions/{subscriptionId}/resourceGroups/{resourceGroup1} --query '{"tenant": tenant, "client_id": appId, "secret": password}'`
    3. **Note** Keep a copy of your subscription_id, client_id, secret and tenant for creating your source in console.redhat.com

3. Create Cost Management source
    1. Follow sources wizard in console.redhat.com
    2. Be sure to create a **storage-only** Azure source type
    3. Visit https://console.redhat.com/api/cost-management/v1/sources/
    4. Find your source and note down its source_uuid, **required** for Azure function

4. Create billing export
    1. See section 1.3 [here](https://access.redhat.com/documentation/en-us/cost_management_service/2023/html/adding_a_microsoft_azure_source_to_cost_management/assembly-adding-azure-sources#configuring-an-azure-daily-export-schedule_adding-an-azure-source) during creation select a storage account or create a new one.
    2. Make a note of the storage account, container name, export dir and export name **Required** for function

5. Setup function to post reports
    1. Create Azure Time Trigger function using [VSCode](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-configuration)
        1. Set the Time trigger function to run once a day

    2. Now you have a basic function created we need to add a few things
        1. Create a **requirements.txt** and add [these](https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/requirements.txt)
        2. Add [Function Code and Queries](#function-code-and-queries),
        3. **NOTE** Be sure to update the required vars
        4. Deploy the function to Azure

    3. Setup blob access for function in Azure portal refer to [this](https://learn.microsoft.com/en-us/samples/azure-samples/functions-storage-managed-identity/using-managed-identity-between-azure-functions-and-azure-storage/)
        1. Navigate to Function App
        2. Select identity in the blade
        3. Turn on System assigned identity
        4. Go to Azure role assignements
        5. Add the following roles for both storage accounts created previously

        ```
        Storage Blob Data Contributor
        Storage Queue Data Contributor
        ```

    4. Optionally store credentials in vault: [Key vault Credentials](#key-vault-credentials)
        1. See [Adding Vault Creds To functions](#adding-vault-creds-to-functions)

## Collecting Finalized Data

1. Setup function to send finalized billing reports
    1. Create Azure Time Trigger function using [VSCode](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-configuration)
        1. Set the Time trigger function to run once a month after the 3rd of the month (after Azure finialises billing data)
        2. Uncomment the following lines

        ```python
        # month_end = now.replace(day=1) - timedelta(days=1)
        # month_start = month_end.replace(day=1)
        # year = month_start.strftime("%Y")
        # month = month_start.strftime("%m")
        # day = month_start.strftime("%d")
        ```

    2. Now you have a basic function created we need to add a few things
        1. Create a **requirements.txt** and add [these](https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/requirements.txt)
        2. Add [Function Code and Queries](#function-code-and-queries),
        3. **NOTE** Be sure to update the required vars
        4. Deploy the function to Azure

    3. Setup blob access for function in Azure portal refer to [this](https://learn.microsoft.com/en-us/samples/azure-samples/functions-storage-managed-identity/using-managed-identity-between-azure-functions-and-azure-storage/)
        1. Navigate to Function App
        2. Select identity in the blade
        3. Turn on System assigned identity
        4. Go to Azure role assignements
        5. Add the following roles for both storage accounts created previously

        ```
        Storage Blob Data Contributor
        Storage Queue Data Contributor
        ```

    4. Optionally store credentials in vault: [Key vault Credentials](#key-vault-credentials)
        1. See [Adding Vault Creds To functions](#adding-vault-creds-to-functions)

## Key vault Credentials

1. Go to *Key Vaults* in Azure
2. Click Create, to create new secret
3. Select the same resource group your function resides in
4. Add Key Vault Name - Next
5. Under *Permissions model* - Choose Vault access policy
6. Within *Access policies* - Create new policy
    1. Select Secret Management from templates
    2. For principal find your Created function
    3. Review and create policy
7. Review and create Vault
8. Click on new vault and select secrets in the blade
9. Generate/import two new secrets
    1. Secret names as follows *ClientIdFromVault* and *ClientSecretFromVault*
    2. Giving them a secret value matching your console.redhat.com service account client_id and client_secret respectively
10. Click on each secret - select the version
11. Copy the secret Identifier URI

## Adding Vault Creds To functions

1. Navigate to your function
2. Select Environment variables under settings in the blade
3. Click add to create variable
4. Name: *ClientIdFromVault*
5. Value: *@Microsoft.KeyVault(SecretUri=YOUR-CLIENT-ID-URI)*
6. Save
7. Add Another application setting for: *ClientSecretFromVault*
8. Value: *@Microsoft.KeyVault(SecretUri=YOUR-CLIENT-SECRET-URI)*
9. Make sure to Replace the URI's with your Secret URI's

## Function Code and Queries

* The default script has the option of Hybrid commited spend or RHEL subscription filtering [azure_function](https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/azure-function.txt)
    * All you need to do is uncomment the relevant line, either `filtered_data = hcs_filtering(df)` OR `filtered_data = rhel_filtering(df)` NOT both
* For custom queries you will need to write your own filtering.
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
    * Once your custom query is built update the custom query in the example script under  **# custom filtering basic example #**.

## Azure filtering required columns

* Below is a list of columns that *MUST* be included and populated for Cost management to process the report correctly
* These columns *MUST* not include spaces, dashes, underscores and *MUST* be in lower case

```
'additionalinfo',
'billingaccountid',
'billingaccountname',
'billingcurrencycode',
'billingperiodenddate',
'billingperiodstartdate',
'chargetype',
'consumedservice',
'costinbillingcurrency',
'date',
'effectiveprice',
'metercategory',
'meterid',
'metername',
'meterregion',
'metersubcategory',
'offerid',
'productname',
'publishername',
'publishertype',
'quantity',
'reservationid',
'reservationname',
'resourcegroup',
'resourceid',
'resourcelocation',
'resourcename',
'servicefamily',
'serviceinfo1',
'serviceinfo2',
'subscriptionid',
'tags',
'unitofmeasure',
'unitprice'
```

* Some of these required columns differ depending on the base report type in use, the example script handles these differences already with the following:

```python
column_translation = {"billingcurrency": "billingcurrencycode", "currency": "billingcurrencycode", "instanceid": "resourceid", "instancename": "resourceid", "pretaxcost": "costinbillingcurrency", "product": "productname", "resourcegroupname": "resourcegroup", "subscriptionguid": "subscriptionid", "servicename": "metercategory", "usage_quantity": "quantity"}
```
