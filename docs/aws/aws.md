# AWS: Customer-filtered data

**Prerequisites:**
- A console.redhat.com service account is required.
- The service account must have at least the `settings.write` permission in the Red Hat Hybrid Console for Cost management access.

For the most part follow [How to schedule athena queries](https://aws.amazon.com/premiumsupport/knowledge-center/schedule-query-athena/)

## Create bucket for reports

1. Create a S3 bucket/With IAM access for Athena results
    1. S3 create new bucket
        1. Create a new S3 bucket with default settings
        2. Save your BUCKET name
    2. IAM Create new Policy
        1. Create policy
        2. Use JSON input and paste the following policy [Bucket-policy](https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/bucket-policy.json)
        3. **Note:** Be sure to replacing CHANGE-ME for your BUCKET name
    3. IAM Create new Role
        1. Create Role
        2. Trusted entity type: AWS Account
        3. Use Another AWS account with the following: `589173575009`
        4. Attach your previously created Policy
        5. Add a Name/Description
        6. Create

2. Create Cost Management source
    1. Follow sources wizard in console.redhat.com
    2. Be sure to create a **storage-only** AWS source type
    3. Visit https://console.redhat.com/api/cost-management/v1/sources/
    4. Find your source and note down its source_uuid, **required** for Lambda scripts

3. Create cost and usage reports
    1. See: [Cost and Usage creation](#cost-and-usage-creation)

4. Configure Athena queries
    1. See: [Configure Athena](#configure-athena)

5. Create correct IAM role/policy for interacting with Lambda/Athena
    1. IAM Create new policy
        1. Use JSON and paste the following lambda/athena policy: [Athena-policy](https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/athena-policy.json)
        2. **Note:** Be sure to update the policy bucket names from CHANGE-ME to match your BUCKET name
        3. Add a Name/Description for the policy and create
    2. IAM create role
        1. Create AWS service role
        2. Use case Lambda
        3. Add policy created above
        4. Add Name/Description of Role
        5. Create

6. Create athena query Lambda function
    1. Create function for querying athena
        1. Author from scratch
        2. Name your function
        3. Select python runtime
        4. Architecture x86_64
        5. Permissions: select role created above
        6. Hit create
    2. Write some code!
        1. Select code tab in the lambda function
        2. Add [Function Code and Athena Queries](#function-code-and-athena-queries), make sure to update the SOURCE_UUID, BUCKET and DATABASE Vars
        3. Hit Deploy then Test and see execution results

7. Create Lambda function to post results
    1. Create function to post report files to Cost Management
        1. Author from scratch
        2. Name your function
        3. Select python runtime
        4. Architecture x86_64
        5. Permissions: select role created above
        6. Hit create
    2. Write some code!
        1. Select code tab in the lambda function
        2. Drop the following [post_code](https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/scripts/post-function.txt) updating the BUCKET, USER, PASS Vars
        3. Using AWS Secrets Manager for credentials: [Secrets manager credentials](#secrets-manager-credentials) Uncomment the following lines and update SECRET_NAME:

        ```
        secret_name = "CHANGEME"
        region_name = "us-east-1"
        secret = get_credentials(secret_name, region_name)
        json_creds = json.loads(secret)
        ```

        4. Hit Deploy then Test and see execution results

8. Create two AmazonEventBridge schedules to trigger the above functions
    1. See [Creating Amazon Eventbridges](#creating-amazon-eventbridges) be sure to update the {CRON_SCHEDULE}
        1. NOTE: The cadence for these should be run once a day.
        2. Example crons: `(0 9 * * ? *)` and `(0 21 * * ? *)`

**GOTCHAS:**

* Why have two functions? - Lambda functions should be simple scripts that run within seconds, however depending on the customers data an athena query may take hours. This enables the customer to easily configure the time between each scripts cron job if extended query time is required.
* The Lambda functions above may hit "errorMessage": ".. Task timed out after 3.04 seconds" Lambda has a default 3s timeout for scripts. On each Lambda function you can change this 3s timeout to 30s if required.

## Collect Finalized Data

1. Create athena query Lambda function
    1. Create function for querying athena
        1. Author from scratch
        2. Name your function
        3. Select python runtime
        4. Architecture x86_64
        5. Permissions: select role created above
        6. Hit create
    2. Write some code for collecting the finalized data
        1. Select code tab in the lambda function
        2. Add [athena_query_function](https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/scripts/athena-query-function.txt), make sure to update the SOURCE_UUID, BUCKET and DATABASE Vars
        3. Note: If you customized your Athena queries from the non stardard ones provided, you will need to do the same here.
        4. Uncomment the following code:

        ```
        # last_month = now.replace(day=1) - timedelta(days=1)
        # year = last_month.strftime("%Y")
        # month = last_month.strftime("%m")
        # day = last_month.strftime("%d")
        # file_name = 'finalized-data.json'
        ```

        5. Hit Deploy then Test and see execution results

2. Create Lambda function to post results
    1. Create function to post report files to Cost Management
        1. Author from scratch
        2. Name your function
        3. Select python runtime
        4. Architecture x86_64
        5. Permissions: select role created above
        6. Hit create
    2. Write some code!
        1. Select code tab in the lambda function
        2. Drop the following [code](https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/scripts/post-function.txt) updating the BUCKET, USER, PASS Vars
        3. Using AWS Secrets Manager for credentials: [Secrets manager credentials](#secrets-manager-credentials) Uncomment the following lines and update SECRET_NAME:

        ```
        secret_name = "CHANGEME"
        region_name = "us-east-1"
        secret = get_credentials(secret_name, region_name)
        json_creds = json.loads(secret)
        ```

        4. Also, make sure to **uncomment** the filename change so that we don't overwrite the daily files.

        ```
        # file_name = 'finalized_data.json'
        ```

        5. Hit Deploy then Test and see execution results

3. Create EventBridge schedule for Cost Mgmt Post function
    1. [Creating Amazon Eventbridges](#creating-amazon-eventbridges) be sure to update the {CRON_SCHEDULE}
        1. NOTE: The cadence for these should be run once a month on or after the 15th of the month since AWS should have finilized your bill by this date.
        2. Example crons: `(0 9 15 * ? *)` and `(0 21 15 * ? *)`

## Cost and Usage creation

1. From the AWS billing console select Data Exports
2. Create Export
3. Select Legacy CUR export
4. Name your export (Save this name for lambda functions)
5. Select Include resource IDs followed by Next
6. Configure S3 bucket to store usage data
7. Set exprt prefix
8. Time Granularity: Hourly
9. Enable export data integration for: Amazon Athena
10. Next to review configuration and Create

## Configure Athena

1. Amazon strongly recommends using CloudFormation and provides instruction on how to do so [here](https://docs.aws.amazon.com/cur/latest/userguide/use-athena-cf.html)
2. Make sure Athena is configured to store query results to the desired S3 bucket see [Querying](https://docs.aws.amazon.com/athena/latest/ug/querying.html)
3. Once Athena is configured you can build a query to filter your data to select lineitems.
4. See [Building an Athena Query](#building-an-athena-query)
5. At this point you can download the query results directly to file from the Athena console, or reference the location of the saved result in S3†

## Secrets Manager Credentials

1. From AWS Secrets Manager - Store a new secret
2. Secret type: Other type of secret
3. Create the following Keys:
    1. client_id
    2. client_secret
4. Populate the values with the appropriate values from your service account
5. Continue to name your secret
6. Continue through and store your secret
7. Update the Role created for your Lambda functions and Include

```json
{
    "Sid": "VisualEditor3",
    "Effect": "Allow",
    "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
    ],
    "Resource": "*"
}
```

## Function Code and Athena Queries

* For standard Hybrid Commited Spend queries use the default [athena_function](https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/scripts/athena-query-function.txt)
    * Initial query to grab all data: **query = f"SELECT * FROM {database}.{export_name} WHERE year = '{year}' AND month = '{month}'"**
    * To filter the data add a **WHERE** clause, for example **WHERE line_item_line_item_description LIKE '%Red Hat%'** would filter out all data that does not have a description containing Red Hat.
    * It's also possible to stack these by using **AND** and **OR** with your **WHERE** clause.
    * Examples:
        1. **service.description** Used to filter to a specifc services such as BigQuery, Cloud Logging etc.
        2. **line_item_product_code**, **line_item_resource_id** or **product_product_name** Used to filter specific product/services based on Code, resource_id or Name.
        3. **product_location**, **product_region** Used to filter data in a specifc region.
        4. **product_product_family**, **product_instance_type** Used to filter resource types by family or specifc instance.
    * Once your custom query is built just replace line 18 with your revised version.

## Creating Amazon Eventbridges

1. Create EventBridge schedule for Athena query function
    1. Add a Name/Description
    2. Select group default
    3. Occurrence: Recurring schedule
    4. Type: Cron-based
    5. Set cron schedule `(example: 0 9 * * ? *)`
    6. Set flexible time window
    7. NEXT
    8. Target detail: AWS Lambda invoke
    9. Select lambda function previously created
    10. NEXT
    11. Enable the schedule
    12. Configure retry logic
    13. Encryption (Ignore)
    14. Permissions: Create new role on the fly
    15. NEXT
    16. Review and create

## Building an Athena Query

* Examples:
    1. **athena_cost_and_usage** This should match your table name in Athena.
    2. **bill_billing_entity** Used to filter specific billing entities.
    3. **line_item_legal_entity** Used to filter specific legal entities.
    4. **line_item_line_item_description** Used to filter on specific line item descriptions.
    5. **year** Used to filter to a specific billing year.
    6. **month** Used to filter on a specific billing month.
* See the following example queries for HCS spend or RHEL ELS subscriptions.
    * The table name **athena_cost_and_usage** MUST be updated to match the name of your configured Athena table.
    * The **year** and **month** can be updated to gather data specific to a particular month.

* The below example query will return filtered data specific for your Hybrid Committed Spend commitment.

```sql
SELECT *
FROM athena_cost_and_usage
WHERE (
        bill_billing_entity = 'AWS Marketplace'
        AND line_item_legal_entity like '%Red Hat%'
    )
    OR (
        line_item_legal_entity like '%Amazon Web Services%'
        AND line_item_line_item_description like '%Red Hat%'
    )
    OR (
        line_item_legal_entity like '%Amazon Web Services%'
        AND line_item_line_item_description like '%RHEL%'
    )
    OR (
        line_item_legal_entity like '%AWS%'
        AND line_item_line_item_description like '%Red Hat%'
    )
    OR (
        line_item_legal_entity like '%AWS%'
        AND line_item_line_item_description like '%RHEL%'
    )
    OR (
        line_item_legal_entity like '%AWS%'
        AND product_product_name like '%Red Hat%'
    )
    OR (
        line_item_legal_entity like '%Amazon Web Services%'
        AND product_product_name like '%Red Hat%'
    )
    AND year = '2024'
    AND month = '07'
```

* The below example query will return filtered data specific to your RHEL subscriptions
    * *Note:* you MUST change **resource_tags_user_tag_column** in the below query to match your RHEL tag column

```sql
SELECT *
FROM athena_cost_and_usage
WHERE (
        line_item_product_code = 'AmazonEC2'
        AND strpos(lower(resource_tags_user_tag_column), 'com_redhat_rhel') > 0
    )
    AND year = '2024'
    AND month = '07'
```

* Query to grab a list of all tagging columns

```sql
SELECT column_name
FROM information_schema.columns
WHERE table_name = 'athena_cost_and_usage'
AND column_name LIKE 'resource_tags_%';
```
