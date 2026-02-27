# GCP: Customer-filtered data

**Prerequisites:**
- A console.redhat.com service account is required.
- The service account must have at least the `settings.write` permission in the Red Hat Hybrid Console for Cost management access.

1. Create new bucket for filtered reports
    1. Follow these [instructions](https://cloud.google.com/storage/docs/creating-buckets)

2. Grant access to bucket data
    1. From IAM & Admin - Roles
    2. Add new role
    3. Add the following permissions:

    ```
    storage.objects.get
    storage.objects.list
    storage.buckets.get
    ```

    4. Create role
    5. Navigate to IAM and add new member
    6. Add the following member: `billing-export@red-hat-cost-management.iam.gserviceaccount.com`
    7. Select the Role created above

3. Create Cost Management source
    1. Follow sources wizard in console.redhat.com
    2. Be sure to create a **storage-only** GCP source type
    3. Add the bucket created in step 1
    4. Visit https://console.redhat.com/api/cost-management/v1/sources/
    5. Find your source and note down its source_uuid, **required** for GCP function

4. Create billing export
    1. In Google Cloud Console, go to BigQuery
    2. In the Explorer panel click on the 3 dot action icon next to your project name and click create dataset
    3. Name your dataset
    4. Click create

5. Setup function to post reports
    1. From Cloud Functions select create function
    2. Name your function
    3. Select HTTP trigger
    4. Optional [Secrets manager credentials](#secrets-manager-credentials)
        1. Within runtime, build, connections, security settings - Go to security
        2. Click reference secret
        3. Select your secret
        4. Set 'Exposed as environment variable'
        5. Select secret version or latest
        6. Click done
        7. Repeat for additional secrets
    5. Hit save and copy your Trigger URL then Next
    6. Select latest supported python runtime
    7. Set Entry Point to **get_filtered_data**
    8. Add [Function Code and Queries](#function-code-and-queries), make sure to update all vars with CHANGE-ME values
    9. Select the requirements.py file and add [this](https://github.com/project-koku/koku-data-selector/blob/main/docs/gcp/scripts/requirements.txt)
    10. Finally hit Deploy

6. Setup cloud scheduler to trigger your function
    1. Navigate to Cloud scheduler
    2. Click schedule a job
    3. Name your schedule
    4. Set frequency to something like: 0 9 * * *
    5. Set timezone and click continue
    6. Paste in your function Trigger URL from above
    7. Add **{"name": "Scheduler"}** to the request body
    8. Set auth header to OIDC token
    9. Select or create a service account with the **Cloud Scheduler Job Runner** AND **Cloud Functions Invoker** roles
    10. Continue and add any retry logic you wish
    11. Hit save

**GOTCHAS:**

* Why do we query a 5 day rolling window? - GCP has a concept of crossover data, essentially you can have billing data for the 1st of a month on the 2nd or 3rd day in a month, this logic means we don't miss that data between queries.
* Why don't we just query a full invoice month? - Another method around crossover data could be to use invoice months, however bigquery requests can be expensive depending on the volume of data, so we want to keep this query range a small as possible to save cost.
* Don't have partitions times in your bigquery dataset table? Remove `DATE(_PARTITIONTIME) BETWEEN DATE('{query_delta}') AND DATE('{now}')` from your query.
* Bigquery error regarding table name? This might be because your table name differs to what we expect and is not interpreted as a string. Try switching `{table_name}` to `'{table_name}'` in your function query.

## Secrets Manager Credentials

1. From GCP Secrets Manager
2. Create two new secrets one for each client_id/client_secret
3. Name your secret
4. Add secret value
5. Hit create

## Function Code and Queries

* For standard Hybrid Committed Spend queries use the default [gcp_function](https://github.com/project-koku/koku-data-selector/blob/main/docs/gcp/scripts/gcp-function.txt)
* For custom queries non HCS we need to edit line 85 in the above function code.
    * Initial query to grab all data: **query = f"SELECT {build_query_select_statement()} FROM {table_name} WHERE DATE(_PARTITIONTIME) BETWEEN '{partition_date_start}' AND {partition_date_end}"**
    * To filter the data add a **WHERE** clause, for example **WHERE service.description LIKE '%Red Hat%'** would filter out all data that does not have a description containing Red Hat.
    * It's also possible to stack these by using **AND** and **OR** with your **WHERE** clause.
    * Examples:
        1. **service.description** Used to filter to a specific services such as BigQuery, Cloud Logging etc.
        2. **project.id**, **project.number** or **project.name** Used to filter specific projects based on ID, Number or Name.
        3. **location.region** Used to filter data in a specific region.
    * You can preview your data in bigquery to help build your desired function query. Once built just replace line 85 with your revised query.

## Final bills

* At the end of the month or rather start of the following month GCP will finish billing for the previous month. At this point we need a mechanism to send these last reports for processing.
* In order to make sure Cost Management has previous month billing accurate we need to create an additional function + scheduled job to trigger it.

1. Setup function to post reports
    1. From Cloud Functions select create function
    2. Name your function
    3. Select HTTP trigger
    4. Optional [Secrets manager credentials](#secrets-manager-credentials)
        1. Within runtime, build, connections, security settings - Go to security
        2. Click reference secret
        3. Select your secret
        4. Set 'Exposed as environment variable'
        5. Select secret version or latest
        6. Click done
        7. Repeat for additional secrets
    5. Hit save and copy your Trigger URL then Next
    6. Select latest supported python runtime
    7. Set Entry Point to **get_filtered_data**
    8. Add [Function Code and Queries](#function-code-and-queries), make sure to update all vars with CHANGE-ME values
    9. Additionally uncomment the following lines

    ```python
    # month_end = now.replace(day=1) - timedelta(days=1)
    # delta = now.replace(day=1) - timedelta(days=query_range)
    # year = month_end.strftime("%Y")
    # month = month_end.strftime("%m")
    # day = month_end.strftime("%d")
    ```

    10. Select the requirements.py file and add [requirements](https://github.com/project-koku/koku-data-selector/blob/main/docs/gcp/scripts/requirements.txt)
    11. Finally hit Deploy

2. Setup cloud scheduler to trigger your function
    1. Navigate to Cloud scheduler
    2. Click schedule a job
    3. Name your schedule
    4. Set frequency to something like: 0 9 4 * * (run on the 4th of every month)
    5. Set timezone and click continue
    6. Paste in your function Trigger URL from above
    7. Add **{"name": "Scheduler"}** to the request body
    8. Set auth header to OIDC token
    9. Select or create a service account with the **Cloud Scheduler Job Runner** AND **Cloud Functions Invoker** roles
    10. Continue and add any retry logic you wish
    11. Hit save
