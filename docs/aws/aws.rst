
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

Configure an S3 bucket
Select Enable report data integration for Amazon Athena 


Configure Athena to query data
==============================

1. Amazon strongly recommends using CloudFormation and provides instruction on how to do so here:
    https://docs.aws.amazon.com/cur/latest/userguide/use-athena-cf.html 
2. Make sure Athena is configured to store query results to the desired S3 bucket
    https://docs.aws.amazon.com/athena/latest/ug/querying.html 
3. Once Athena is configured the following query will return the filtered dataset specific to your Red Hat commitment. The table name following the FROM keyword would be updated to match the name of the table configured in your Athena instance. The year and month can be updated to gather data specific to a particular month.


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
5. POST to our (newly developed) cost management API endpoint with file location/path of the query results for ingestion by Cost Management

========================
Lamda/Athena Query Setup
========================

Certain portions of the above process could be automated to further reduce the need for manual querying. 

Instead of manually kicking off a query in the Athena console and then having a user POST to the cost management API endpoint, a serverless function could be written to do this on a configurable schedule. Using Amazon Lambda (https://aws.amazon.com/lambda/), a function could be written to run the above Athena query, gather the result file name and location and send the POST message to the cost management API, whereupon Cost MAnagement would consume the data. 


Lamda/Athena Setup
==================
For the most part follow this: https://aws.amazon.com/premiumsupport/knowledge-center/schedule-query-athena/

1. Create correct IAM role/policy for interacting with Lambda/Athena
    a. IAM Create new policy
        i. Use JSON and paste the following lambda/athena policy: `athena-policy file`.
        ii. **Note:** Be sure to update the policy bucket names from CHANGE-ME to match your CUR bucket
        iii. Add a Name/Description for the policy and create
    b. IAM create role
        i. Create AWS service role
        ii. Use case Lambda
        iii. Add policy created above
        iv. Add Name/Description of Role
        v. Create
2. Create/setup Lambda function
    a. Create new Lambda function
        i. Author from scratch
        ii. Name your function
        iii. Select python runtime
        iv. Architecture x86_64
        v. Permissions: select role created above
        vi. Hit create
    b. Write some code
        i. Select code tab in the lambda function
        ii. Drop the following code updating the DATABASE and BUCKET Vars:

            import boto3
            import uuid
            from datetime import datetime

            BUCKET = 'CHANGE-ME'
            now = datetime.now()
            year = now.strftime("%Y")
            month = now.strftime("%m")
            day = now.strftime("%d")

            # Database to execute the query against
            DATABASE = 'CHANGE-ME'

            # Output location for query results
            output=f's3://{BUCKET}/{year}/{month}/{day}/{uuid.uuid4()}'

            # Query string to execute
            query = f"SELECT * FROM {DATABASE}.athena_cost_and_usage WHERE ((bill_billing_entity = 'AWS Marketplace' AND line_item_legal_entity like '%Red Hat%') OR (line_item_legal_entity like '%Amazon Web Services%' AND line_item_line_item_description like '%Red Hat%')) AND year = '{year}' AND month = '{month}'"

            def lambda_handler(event, context):
                # Initiate the Boto3 Client
                client = boto3.client('athena')

                # Start the query execution
                response = client.start_query_execution(
                    QueryString=query,
                    QueryExecutionContext={
                        'Database': DATABASE
                    },
                    ResultConfiguration={
                        'OutputLocation': output
                    }
                )

                # Return response after starting the query execution, database querying against and output dir for query results
                return response, DATABASE, output

        iii. **Note:** Be sure to update the Bucket and Database names
        iv. Hit Deploy then Test and see execution result

3. Schedule the function to run using AmazonEventBridge
    a. Create EventBridge schedule
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

GOTCHAS:
Today this Lambda function just triggers a query in Athena and is not aware when the query is complete. This mean it cannot POST the file locations to Cost Management.
