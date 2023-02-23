
=============================
GCP: Customer-filtered data
=============================

Prerequisite
============

1. Create billing export, see `here <https://access.redhat.com/documentation/en-us/cost_management_service/2022/html/adding_a_google_cloud_source_to_cost_management/assembly-adding-gcp-sources>`_

=======================
Automate Filtered Query
=======================

GCP bucket store
================

1. Create new bucket for filtered reports 
    a. Follow this `instructions <https://cloud.google.com/storage/docs/creating-buckets>`_

2. Grant access to bucket data
    a. Add `billing-export@red-hat-cost-management.iam.gserviceaccount.com` giving the 'Storage Object Viewer' role

3. Create Cost Management source
    a. Follow sources wizard in console.redhat.com
    b. Be sure to create a **storage-only** GCP source type
    c. Add the bucket created in step 1
    d. Visit https://console.redhat.com/api/cost-management/v1/sources/
    e. Find your source and note down its source_uuid, **required** for GCP function

4. Setup function to post reports
    a. From Cloud Functions select create function
    b. Name your function
    c. Select HTTP trigger
    d. Hit save and copy your Trigger URL then Next
    e. Select python 3.9 runtime
    f. Set Entry Point to **get_filtered_data**
    g. Paste in the following code, make sure to update all vars with CHANGE-ME values

    .. code-block::

        import csv
        import datetime
        import logging
        import uuid
        import requests
        from google.cloud import bigquery
        from google.cloud import storage
        from itertools import islice
        from dateutil.relativedelta import relativedelta
        from tempfile import NamedTemporaryFile

        now = datetime.datetime.now()
        year = now.strftime("%Y")
        month = now.strftime("%m")
        day = now.strftime("%d")
        report_prefix=f"{year}/{month}/{day}/{uuid.uuid4()}"
        partition_date = f"{year}-{month}-{day}"

        scan_start = now - relativedelta(days=10)
        scan_end = now

        # Required vars to update
        SOURCE_UUID = "CHANGE-ME"               # Cost management source_uuid
        USER = "CHANGE-ME"                      # Cost management username
        PASS = "CHANGE-ME"                      # Cost management password
        BUCKET = "CHANGE-ME"                    # Filtered data GCP Bucket
        PROJECT_ID = "CHANGE-ME"                # Your project ID
        DATASET = "CHANGE-ME"                   # Your dataset name
        TABLE_ID = "CHANGE-ME"                  # Your table ID

        gcp_big_query_columns = [
            "billing_account_id",
            "service.id",
            "service.description",
            "sku.id",
            "sku.description",
            "usage_start_time",
            "usage_end_time",
            "project.id",
            "project.name",
            "project.labels",
            "project.ancestry_numbers",
            "labels",
            "system_labels",
            "location.location",
            "location.country",
            "location.region",
            "location.zone",
            "export_time",
            "cost",
            "currency",
            "currency_conversion_rate",
            "usage.amount",
            "usage.unit",
            "usage.amount_in_pricing_units",
            "usage.pricing_unit",
            "credits",
            "invoice.month",
            "cost_type",
            "resource.name",
            "resource.global_name",
        ]
        table_name = ".".join([PROJECT_ID, DATASET, TABLE_ID])

        BATCH_SIZE = 200000

        def batch(iterable, n):
            """Yields successive n-sized chunks from iterable"""
            it = iter(iterable)
            while chunk := tuple(islice(it, n)):
                yield chunk

        def build_query_select_statement():
            """Helper to build query select statement."""
            columns_list = gcp_big_query_columns.copy()
            columns_list = [
                f"TO_JSON_STRING({col})" if col in ("labels", "system_labels", "project.labels") else col
                for col in columns_list
            ]
            # Swap out resource columns with NULLs when we are processing
            # a non-resource-level BigQuery table
            columns_list = [
                f"NULL as {col.replace('.', '_')}"
                if col in ("resource.name", "resource.global_name")
                and "resource" not in TABLE_ID
                else col
                for col in columns_list
            ]
            columns_list.append("DATE(_PARTITIONTIME) as partition_date")
            return ",".join(columns_list)
            
        def create_reports():
            query = f"SELECT {build_query_select_statement()} FROM {table_name} WHERE DATE(_PARTITIONTIME) = '{partition_date}' AND sku.description LIKE '%RedHat%' OR sku.description LIKE '%Red Hat%' OR  service.description LIKE '%Red Hat%'"
            client = bigquery.Client()
            query_job = client.query(query).result()
            column_list = gcp_big_query_columns.copy()
            column_list.append("partition_date")
            files_list = []
            storage_client = storage.Client()
            bucket = storage_client.bucket(BUCKET)
            for i, rows in enumerate(batch(query_job, BATCH_SIZE)):
                csv_file = f"{report_prefix}/{partition_date}_{str(i)}.csv"
                files_list.append(csv_file)
                blob = bucket.blob(csv_file)
                with blob.open(mode='w') as f:
                    writer = csv.writer(f)
                    writer.writerow(column_list)
                    writer.writerows(rows)
            return files_list

        def post_data(files_list):
            # Post CSV's to console.redhat.com API
            url = "https://console.redhat.com/api/cost-management/v1/ingress/reports/"
            data = {"source": SOURCE_UUID, "reports_list": files_list, "bill_year": year, "bill_month": month}
            resp = requests.post(url, data=data, auth=(USER, PASS))
            return resp

        def get_filtered_data(request):
            files_list = create_reports()
            resp = post_data(files_list)
            return f'Files posted! {resp}'


    h. Select the requirements.py file and paste the following

    .. code-block::

        # Function dependencies, for example:
        # package>=version
        requests
        google-cloud-bigquery
        google-cloud-storage

    i. Finally hit Deploy

5. Setup cloud scheduler to trigger your function
    a. Navigate to Cloud scheduler
    b. Click schedule a job
    c. Name your schedule
    d. Set frequency to something like: 0 9 * * *
    e. Set timezone and click continue
    f. Paste in your function Trigger URL from above
    g. Add **{"name": "Scheduler"}** to the request body
    h. Set auth header to OIDC token
    i. Select or create a service account with the **Cloud Scheduler Job Runner** AND **Cloud Functions Invoker** roles
    j. Continue and add any retry logic you wish
    k. Hit save


**GOTCHAS:**