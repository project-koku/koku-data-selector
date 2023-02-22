
=============================
Azure: Customer-filtered data
=============================

Prerequisite
============

1. Create billing export, see `here <https://access.redhat.com/documentation/en-us/cost_management_service/2022/html/adding_a_microsoft_azure_source_to_cost_management/assembly-adding-azure-sources>`_ 
2. Make a note of the storage account in use for the cost export **Required** for function

=======================
Automate Filtered Query
=======================

Azure resource group and storage account
========================================

1. Create new resource group and storage account for filtered reports using these `instructions <https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal>`_

2. Setup read access for Cost Management
    a. In Azure Cloud Shell run the following to obtain the subscription ID: ``az account show --query "{subscription_id: id }"``
    b. Create Resource group role: ``az ad sp create-for-rbac -n "CostManagement" --role "Storage Account Contributor"  --scope /subscriptions/{subscriptionId}/resourceGroups/{resourceGroup1} --query '{"tenant": tenant, "client_id": appId, "secret": password}'``
    c. **Note** Keep a copy of your subscription_id, client_id, secret and tenant for creating your source in console.redhat.com

3. Create Cost Management source
    a. Folow sources wizard in console.redhat.com
    b. Be sure to create a **storage-only** Azure source type
    c. Visit https://console.redhat.com/api/cost-management/v1/sources/
    d. Find your source and note down its source_uuid, **required** for Azure function

4. Setup function to post reports
    a. Create Azure Time Trigger function using `VSCode <https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=nodejs#debugging-functions-locally>`_
        i. Set the Time trigger function to run once a day

    b. Now you have a basic function created we need to add a few things
        i. Create a **requirements.txt** and add the following:

        .. code-block::

            pandas
            azure-identity
            azure-storage-blob

        ii. Replace the Trigger function code with the following:

        .. code-block::

            import datetime
            import logging
            import uuid
            import requests
            import pandas as pd
            from azure.identity import DefaultAzureCredential
            from azure.storage.blob import BlobServiceClient, ContainerClient

            import azure.functions as func


            def main(mytimer: func.TimerRequest) -> None:
                utc_timestamp = datetime.datetime.utcnow().replace(
                    tzinfo=datetime.timezone.utc).isoformat()

                default_credential = DefaultAzureCredential()

                now = datetime.datetime.now()
                year = now.strftime("%Y")
                month = now.strftime("%m")
                day = now.strftime("%d")
                output_blob_name=f"{year}/{month}/{day}/{uuid.uuid4()}.csv"

                # Required vars to update
                source_uuid = "CHANGE-ME"               # Cost management source_uuid
                USER = "CHANGE-ME"                      # Cost management username
                PASS = "CHANGE-ME"                      # Cost management password
                cost_export_store = "CHANGE-ME"         # Cost export storage account url 
                cost_export_container = "CHANGE-ME"     # Cost export container
                filtered_data_account_url = "CHANGE-ME" # Filtered data storage account url
                filtered_data_container = "CHANGE-ME"   # Filtered data container

                # Create the BlobServiceClient object
                blob_service_client = BlobServiceClient(filtered_data_account_url, credential=default_credential)
                container_client = ContainerClient(cost_export_store, credential=default_credential, container_name=cost_export_container)

                blob_list = container_client.list_blobs()
                latest_blob = None
                for blob in blob_list:
                    if latest_blob:
                        if blob.last_modified > latest_blob.last_modified:
                            latest_blob = blob
                    else:
                        latest_blob = blob

                bc = container_client.get_blob_client(blob=latest_blob) # DO we need this? Or can this work latest_blob.download_blob()
                data = bc.download_blob()
                blobjct = "/tmp/blob.csv"
                with open(blobjct, "wb") as f:
                    data.readinto(f)
                df = pd.read_csv(blobjct)

                filtered_data = df.loc[((df["publisherType"] == "Marketplace") & (df["publisherName"].astype(str).str.contains("Red Hat"))) | ((df["publisherName"] == "Microsoft") & (df['meterSubCategory'].astype(str).str.contains("Red Hat") | df['serviceInfo2'].astype(str).str.contains("Red Hat")))]

                filtered_data_csv = filtered_data.to_csv (index_label="idx", encoding = "utf-8")

                blob_client = blob_service_client.get_blob_client(container=filtered_data_container, blob=output_blob_name)

                blob_client.upload_blob(filtered_data_csv, overwrite=True)
                
                # Post results to console.redhat.com API
                url = "https://console.redhat.com/api/cost-management/v1/ingress/reports/"
                data = {"source": source_uuid, "reports_list": [f"{filtered_data_container}/{output_blob_name}"], "bill_year": year, "bill_month": month}
                resp = requests.post(url, data=data, auth=(USER, PASS))
                logging.info(f'Post result: {resp}')

                if mytimer.past_due:
                    logging.info('The timer is past due!')

                logging.info('Python timer trigger function ran at %s', utc_timestamp)

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