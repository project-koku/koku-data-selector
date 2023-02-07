
===========================
AWS: Customer-filtered data
===========================

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


Configure Athena to query data
==============================

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
        AND year = '2022'
        AND month = '10'

4. At this point you can download the query results directly to file from the Athena console, or reference the location of the saved result in S3†
5. POST to our *newly developed* cost management API endpoint with file location/path of the query results for ingestion by Cost Management




=====================
Automate Athena Query
=====================

Certain portions of the above process could be automated to further reduce the need for manual querying. 

Instead of manually kicking off a query in the Athena console and then having a user POST to the cost management API endpoint, a serverless function could be written to do this on a configurable schedule. Using Amazon `Lambda <https://aws.amazon.com/lambda/>`_, a function could be written to run the above Athena query, gather the result file name and location and send the POST message to the cost management API, whereupon Cost Management would consume the data. 


Lamda/Athena Setup
==================
For the most part follow `How to schedule athena queries <https://aws.amazon.com/premiumsupport/knowledge-center/schedule-query-athena/>`_

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

2. Create correct IAM role/policy for interacting with Lambda/Athena
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
3. Create athena query Lambda function
    a. Create function for querying athena
        i. Author from scratch
        ii. Name your function
        iii. Select python runtime
        iv. Architecture x86_64
        v. Permissions: select role created above
        vi. Hit create
    b. Write some code!
        i. Select code tab in the lambda function
        ii. Drop the following code below updating the DATABASE and BUCKET Vars
        iii. Hit Deploy then Test and see execution results


.. code-block::

    import boto3
    import uuid
    import json
    from datetime import datetime

    now = datetime.now()
    year = now.strftime("%Y")
    month = now.strftime("%m")
    day = now.strftime("%d")

    # Vars to Change!
    source_uuid = "CHANGEME"                                    # Cost Management source_uuid
    bucket = 'CHANGEME'                                         # Bucket created for query results
    database = 'athenacurcfn_athena_cost_and_usage'             # Database to execute athena queries
    output=f's3://{bucket}/{year}/{month}/{day}/{uuid.uuid4()}' # Output location for query results

    # Athena query
    query = f"SELECT * FROM {database}.athena_cost_and_usage WHERE ((bill_billing_entity = 'AWS Marketplace' AND line_item_legal_entity like '%Red Hat%') OR (line_item_legal_entity like '%Amazon Web Services%' AND line_item_line_item_description like '%Red Hat%')) AND year = '{year}' AND month = '{month}'"

    def lambda_handler(event, context):
        # Initiate Boto3 athena Client
        athena_client = boto3.client('athena')
        
        # Trigger athena query
        response = athena_client.start_query_execution(
            QueryString=query,
            QueryExecutionContext={
                'Database': database
            },
            ResultConfiguration={
                'OutputLocation': output
            }
        )
        
        # Save query execution to s3 object
        s3 = boto3.client('s3')
        json_object = {"source_uuid": source_uuid, "bill_year": year, "bill_month": month, "query_execution_id": response.get("QueryExecutionId"), "result_prefix": output}
        s3.put_object(
            Body=json.dumps(json_object),
            Bucket=bucket,
            Key='query-data.json'
        )
        
        return json_object


4. Create Lambda function to post results
    a. Create function to post report files to Cost Management
        i. Author from scratch
        ii. Name your function
        iii. Select python runtime
        iv. Architecture x86_64
        v. Permissions: select role created above
        vi. Hit create
    b. Write some code!
        i. Select code tab in the lambda function
        ii. Drop the following code below updating the BUCKET, USER, PASS Vars
        iii. Hit Deploy then Test and see execution results

.. code-block::

    import boto3
    import json
    import requests

    bucket = "CHANGEME"  # Bucket for athena query results
    USER = "CHANGEME"    # Cost Management Username
    PASS = "CHANGEME"    # Cost Management Password

    def lambda_handler(event, context):
        # Initiate Boto3 s3 and fetch query file
        s3_resource = boto3.resource('s3')
        json_content = json.loads(s3_resource.Object(bucket, 'query-data.json').get()['Body'].read().decode('utf-8'))
        
        # Initiate Boto3 athena Client and attempt to fetch athena results
        athena_client = boto3.client('athena')
        try:
            athena_results = athena_client.get_query_execution(QueryExecutionId=json_content["query_execution_id"])
        except Exception as e:
            return f"Error fetching athena query results: {e} \n Consider increasing the time between running and fetching results"

        reports_list = []
        prefix = json_content["result_prefix"].split(f'{bucket}/')[-1]
        
        # Initiate Boto3 s3 client
        s3_client = boto3.client('s3')
        result_data = s3_client.list_objects(Bucket=bucket, Prefix=prefix)
        for item in result_data.get("Contents"):
            if item.get("Key").endswith(".csv"):
                print(item.get("Key"))
                reports_list.append(item.get("Key"))
                
        # Post results to console.redhat.com API
        url = "https://console.redhat.com/api/cost-management/v1/ingress/reports/"
        data = {"source": json_content["source_uuid"], "reports_list": reports_list, "bill_year": json_content["bill_year"], "bill_month": json_content["bill_month"]}
        resp = requests.post(url, data=data, auth=(USER, PASS))

        return resp


5. Create two AmazonEventBridge schedules to trigger the above functions
    a. Create EventBridge schedule for Athena query function
        i. Add a Name/Description
        ii. Select group default
        iii. Occurrence: Recurring schedule
        iv. Type: Cron-based
        v. Set cron schedule **(0 9 * * ? *)** This will be 9AM Every day
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
    b. Create EventBridge schedule for Cost Mgmt Post function
        i. Add a Name/Description
        ii. Select group default
        iii. Occurrence: Recurring schedule
        iv. Type: Cron-based
        v. Set cron schedule **(0 21 * * ? *)** This will be 9PM Every day
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

**GOTCHAS:**

* Why have two functions? - Lambda functions should be simple scripts that run within seconds, however depending on the customers data an athena query may take hours. This enables the customer to easily configure the time between each scripts cron job if extended query time is required.
* The Lambda functions above may hit "errorMessage": ".. Task timed out after 3.04 seconds" Lambda has a default 3s timeout for scripts. On each Lambda function you can change this 3s timeout to 30s if required.

