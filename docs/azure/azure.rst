
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
    a. See sections 1.1 and 1.3 `here <https://access.redhat.com/documentation/en-us/cost_management_service/2022/html/adding_a_microsoft_azure_source_to_cost_management/assembly-adding-azure-sources>`_ NOTE: Since Cost Managment does not need direct access you can skip 1.2 
    b. Make a note of the storage account in use for the cost export **Required** for function


5. Setup function to post reports
    a. Create Azure Time Trigger function using `VSCode <https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=nodejs#debugging-functions-locally>`_
        i. Set the Time trigger function to run once a day

    b. Now you have a basic function created we need to add a few things
        i. Create a **requirements.txt** and add `this <https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/requirements.txt>`_
        ii. Replace the Trigger function code with `this <https://github.com/project-koku/koku-data-selector/blob/main/docs/azure/scripts/azure-function.txt>`_
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

**GOTCHAS:**