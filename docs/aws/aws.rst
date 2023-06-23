
===========================
AWS: Customer-filtered data
===========================
For the most part follow `How to schedule athena queries <https://aws.amazon.com/premiumsupport/knowledge-center/schedule-query-athena/>`_


Create bucket for reports
=========================

1. Create a S3 bucket/With IAM access for Athena results
    a. S3 create new bucket
        i. Create a new S3 bucket with default settings
        ii. Save your BUCKET name
    b. IAM Create new Policy
        i. Create policy
        ii. Use JSON input and paste the following policy `Bucket-policy <https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/bucket-policy.rst>`_
        iii. **Note:** Be sure to replacing CHANGE-ME for your BUCKET name
    c. IAM Create new Role
        i. Create Role
        ii. Trusted entity type: AWS Account
        iii. Use Another AWS account with the following: ``589173575009``
        iv. Attach your previously created Policy
        v. Add a Name/Description
        vi. Create

2. Create Cost Management source
    a. Follow sources wizard in console.redhat.com
    b. Be sure to create a **storage-only** AWS source type
    c. Visit https://console.redhat.com/api/cost-management/v1/sources/
    d. Find your source and note down its source_uuid, **required** for Lambda scripts

3. Create cost and usage reports
    a. See: `Cost and Usage creation`_

4. Configure Athena queries
    a. See: `Configure Athena`_

5. Create correct IAM role/policy for interacting with Lambda/Athena
    a. IAM Create new policy
        i. Use JSON and paste the following lambda/athena policy: `Athena-policy <https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/athena-policy.rst>`_
        ii. **Note:** Be sure to update the policy bucket names from CHANGE-ME to match your BUCKET name
        iii. Add a Name/Description for the policy and create
    b. IAM create role
        i. Create AWS service role
        ii. Use case Lambda
        iii. Add policy created above
        iv. Add Name/Description of Role
        v. Create

6. Create athena query Lambda function
    a. Create function for querying athena
        i. Author from scratch
        ii. Name your function
        iii. Select python runtime
        iv. Architecture x86_64
        v. Permissions: select role created above
        vi. Hit create
    b. Write some code!
        i. Select code tab in the lambda function
        ii. Add `Function Code and Athena Queries`_, make sure to update the SOURCE_UUID, BUCKET and DATABASE Vars
        iii. Hit Deploy then Test and see execution results

7. Create Lambda function to post results
    a. Create function to post report files to Cost Management
        i. Author from scratch
        ii. Name your function
        iii. Select python runtime
        iv. Architecture x86_64
        v. Permissions: select role created above
        vi. Hit create
    b. Write some code!
        i. Select code tab in the lambda function
        ii. Drop the following `post_code <https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/scripts/post-function.txt>`_ updating the BUCKET, USER, PASS Vars
        iii. Using AWS Secrets Manager for credentials: `Secrets manager credentials`_ Uncomment the following lines and update SECRET_NAME:

        .. code-block::

            secret_name = "CHANGEME"
            region_name = "us-east-1"
            secret = get_credentials(secret_name, region_name)
            json_creds = json.loads(secret)

        iv. Hit Deploy then Test and see execution results

8. Create two AmazonEventBridge schedules to trigger the above functions
    a. See `Creating Amazon Eventbridges`_ be sure to update the {CRON_SCHEDULE}
        i. NOTE: The cadence for these should be run once a day.
        ii. Example crons: (0 9 * * ? *) and (0 21 * * ? *)

**GOTCHAS:**

* Why have two functions? - Lambda functions should be simple scripts that run within seconds, however depending on the customers data an athena query may take hours. This enables the customer to easily configure the time between each scripts cron job if extended query time is required.
* The Lambda functions above may hit "errorMessage": ".. Task timed out after 3.04 seconds" Lambda has a default 3s timeout for scripts. On each Lambda function you can change this 3s timeout to 30s if required.


Collect Finalized Data
======================
1. Create athena query Lambda function
    a. Create function for querying athena
        i. Author from scratch
        ii. Name your function
        iii. Select python runtime
        iv. Architecture x86_64
        v. Permissions: select role created above
        vi. Hit create
    b. Write some code for collecting the finalized data
        i. Select code tab in the lambda function
        ii. Add `athena_query_function <https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/scripts/athena-query-function.txt>`_ , make sure to update the SOURCE_UUID, BUCKET and DATABASE Vars
        iii. Note: If you customized your Athena queries from the non stardard ones provided, you will need to do the same here.
        iv. Uncomment the following code:

        .. code-block::

            # last_month = now.replace(day=1) - timedelta(days=1)
            # month = last_month.strftime("%m")
            # day = last_month.strftime("%d")
            # file_name = 'finalized-data.json'

        iv. Hit Deploy then Test and see execution results
2. Create Lambda function to post results
    a. Create function to post report files to Cost Management
        i. Author from scratch
        ii. Name your function
        iii. Select python runtime
        iv. Architecture x86_64
        v. Permissions: select role created above
        vi. Hit create
    b. Write some code!
        i. Select code tab in the lambda function
        ii. Drop the following `code <https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/scripts/post-function.txt>`_ updating the BUCKET, USER, PASS Vars
        iii. Using AWS Secrets Manager for credentials: `Secrets manager credentials`_ Uncomment the following lines and update SECRET_NAME:

        .. code-block::

            secret_name = "CHANGEME"
            region_name = "us-east-1"
            secret = get_credentials(secret_name, region_name)
            json_creds = json.loads(secret)

        iv. Also, make sure to **uncomment** the filename change so that we don't overwrite the daily files.

        .. code-block::

            # file_name = 'finalized_data.json'

        iv. Hit Deploy then Test and see execution results
3. Create EventBridge schedule for Cost Mgmt Post function
    a. `Creating Amazon Eventbridges`_ be sure to update the {CRON_SCHEDULE}
        i. NOTE: The cadence for these should be run once a month on or after the 15th of the month since AWS should have finilized your bill by this date.
        ii. Example crons: (0 9 15 * ? *) and (0 21 15 * ? *)



Cost and Usage creation
=======================

1. From the AWS billing console select Cost & usage reports
2. Create report
3. Name your report
4. Select Include resource IDs followed by Next
5. Configure S3 bucket to store usage data
6. Set report prefix
7. Time Granularity: Hourly
8. Enable report data integration for: Amazon Athena
9. Next to review configuration and Create


Configure Athena
================

1. Amazon strongly recommends using CloudFormation and provides instruction on how to do so `here <https://docs.aws.amazon.com/cur/latest/userguide/use-athena-cf.html>`_
2. Make sure Athena is configured to store query results to the desired S3 bucket see `Querying <https://docs.aws.amazon.com/athena/latest/ug/querying.html>`_
3. Once Athena is configured the following query will return the filtered dataset specific to your Red Hat commitment. The table name following the FROM keyword would be updated to match the name of the table configured in your Athena instance. The year and month can be updated to gather data specific to a particular month.

.. code-block::

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
        AND year = '2022'
        AND month = '10'

4. At this point you can download the query results directly to file from the Athena console, or reference the location of the saved result in S3†


Secrets Manager Credentials
===========================

1. From AWS Secrets Manager - Store a new secret
2. Secret type: Other type of secret
3. Create the following Keys:
    i. username
    ii. password
4. Populate the values with the appropriate username/password
5. Name your secret
6. Continue through and store your secret
7. Update the Role created for your Lambda functions and Include

.. code-block::

    {
        "Sid": "VisualEditor3",
        "Effect": "Allow",
        "Action": [
            "secretsmanager:GetSecretValue",
            "secretsmanager:DescribeSecret"
        ],
        "Resource": "*"
    }


Function Code and Athena Queries
================================
* For standard Hybrid Commited Spend queries use the default `athena_function <https://github.com/project-koku/koku-data-selector/blob/main/docs/aws/scripts/athena-query-function.txt>`_
* For custom queries non HCS we need to edit line 18 in the above function code.
    * Initial query to grab all data: **query = f"SELECT * FROM {database}.koku_athena WHERE year = '{year}' AND month = '{month}'"**
    * To filter the data add a **WHERE** clause, for example **WHERE line_item_line_item_description LIKE '%Red Hat%'** would filter out all data that does not have a description containing Red Hat.
    * It's also possible to stack these by using **AND** and **OR** with your **WHERE** clause.
    * Examples:
        1. **service.description** Used to filter to a specifc services such as BigQuery, Cloud Logging etc.
        2. **line_item_product_code**, **line_item_resource_id** or **product_product_name** Used to filter specific product/services based on Code, resource_id or Name.
        3. **product_location**, **product_region** Used to filter data in a specifc region.
        4. **product_product_family**, **product_instance_type** Used to filter resource types by family or specifc instance.
    * Once your custom query is built just replace line 18 with your revised version.


Creating Amazon Eventbridges
============================
a. Create EventBridge schedule for Athena query function
    i. Add a Name/Description
    ii. Select group default
    iii. Occurrence: Recurring schedule
    iv. Type: Cron-based
    v. Set cron schedule **{CRON_SCHEDULE}** This will be 9AM Every day
    vi. Set flexible time window
    vii. NEXT
    viii. Target detail: AWS Lambda invoke
    ix. Select lambda function previously created
    x. NEXT
    xi. Enable the schedule
    xii. Configure retry logic
    xiii. Encryption (Ignore)
    xiv. Permissions: Create new role on the fly
    xv. NEXT
    xvi. Review and create
