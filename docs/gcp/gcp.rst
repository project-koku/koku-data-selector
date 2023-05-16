
===========================
GCP: Customer-filtered data
===========================

GCP bucket store
================

1. Create new bucket for filtered reports 
    a. Follow this `instructions <https://cloud.google.com/storage/docs/creating-buckets>`_

2. Grant access to bucket data
    a. From IAM & Admin - Roles
    b. Add new role
    c. Add the following permissions:

    .. code::

        storage.objects.get
        storage.objects.list
        storage.buckets.get

    d. Create role
    e. Navigate to IAM and add new member
    f. Add the following member: `billing-export@red-hat-cost-management.iam.gserviceaccount.com`
    g. Select the Role created above

3. Create Cost Management source
    a. Follow sources wizard in console.redhat.com
    b. Be sure to create a **storage-only** GCP source type
    c. Add the bucket created in step 1
    d. Visit https://console.redhat.com/api/cost-management/v1/sources/
    e. Find your source and note down its source_uuid, **required** for GCP function

4. Create billing export
    a. see sections 1.4 and 1.5 `here <https://access.redhat.com/documentation/en-us/cost_management_service/2023/html/adding_a_google_cloud_source_to_cost_management/assembly-adding-gcp-sources#creating-a-dataset-gcp_adding-gcp-sources>`_

5. Setup function to post reports
    a. From Cloud Functions select create function
    b. Name your function
    c. Select HTTP trigger
    d. Optional `Secrets manager credentials`_
        i. Within runtime, build, connections, security settings - Go to security
        ii. Click reference secret
        iii. Select your secret
        iv. Set 'Exposed as environment variable'
        v. Select secret version or latest
        vi. Click done
        vii. Repeat for additional secrets
    e. Hit save and copy your Trigger URL then Next
    f. Select python 3.9 runtime
    g. Set Entry Point to **get_filtered_data**
    h. Add `Function Code and Queries`_, make sure to update all vars with CHANGE-ME values
    i. Select the requirements.py file and add `this <https://github.com/project-koku/koku-data-selector/blob/main/docs/gcp/scripts/requirements.txt>`_
    j. Finally hit Deploy

6. Setup cloud scheduler to trigger your function
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

* Why do we query a 5 day rolling window? - GCP has a concept of crossover data, essentially you can have billing data for the 1st of a month on the 2nd or 3rd day in a month, this logic means we don't miss that data between queries.
* Why don't we just query a full invoice month? - Another method around crossover data could be to use invice months, however bigquery requests can be expensive depending on the volume of data, so we want to keep this query range a small as possible to save cost.
* Don't have partitions in your bigquery dataset table? Switch to using `Non-partitions-function <https://github.com/project-koku/koku-data-selector/blob/main/docs/gcp/scripts/gcp-function-non-partition-dates.txt>`_
* Bigquery error regarding table name? This might be because your table name differs to what we expect and is not interpreted as a string. Try switching `{table_name}` to `'{table_name}'` in your function query.

Secrets Manager Credentials
===========================

1. From GCP Secrets Manager 
2. Create two new secrets one for username/password
3. Name your secret
4. Add secret value
5. Hit create

Function Code and Queries
=========================
* For standard Hybrid Commited Spend queries use the default `gcp_function <https://github.com/project-koku/koku-data-selector/blob/main/docs/gcp/scripts/gcp-function.txt>`_
* For custom queries non HCS we need to edit line 85 in the above function code.
    * Initial query to grab all data: **query = f"SELECT {build_query_select_statement()} FROM {table_name} WHERE DATE(_PARTITIONTIME) BETWEEN '{partition_date_start}' AND {partition_date_end}"**
    * To filter the data add a **WHERE** clause, for example **WHERE service.description LIKE '%Red Hat%'** would filter out all data that does not have a description containing Red Hat.
    * It's also possible to stack these by using **AND** and **OR** with your **WHERE** clause.
    * Examples:
        1. **service.description** Used to filter to a specifc services such as BigQuery, Cloud Logging etc.
        2. **project.id**, **project.number** or **project.name** Used to filter specific projects based on ID, Number or Name.
        3. **location.region** Used to filter data in a specifc region.
    * You can preview your data in bigquery to help build your desired function query. Once built just replace line 85 with your revised query.
