# Orchestrate-Data-Lake-and-Redshift-with-Step-Functions
In this lab, we'll show you how to use AWS Step Functions to orchestrate actions in your Amazon Redshift warehouse using the Redshift Data API.  We will create use a Step Function model to catalog and prepare the data, load into Amazon Redshift and create a view spanning the warehouse and data lake with Redshift Spectrum.

:information_source: You will run this lab in your own AWS account and running this lab will incur some costs. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Overview

In this lab, we will use AWS Step Functions to orchestrate an end to end data pipeline. We will start by configuring our data lake in Lake Formation, load some data into an S3 bucket, then transform and catalog the data using AWS Glue. After that, we'll use the Redshift data API to load a portion of the data into Redshift, and create a view joining "warm" data in Amazon Redshift with "cold" data stored in our data lake. Finally, we'll use the UNLOAD command to send aggregated statistics from the Redshift warehouse back to the data lake and use Amazon Athena to run ad-hoc queries on the results.

This lab will use the following services:
- AWS Step Functions - a low-code, visual workflow service that developers use to build distributed applications, automate IT and business processes, and build data and machine learning pipelines using AWS services. Workflows manage failures, retries, parallelization, service integrations, and observability so developers can focus on higher-value business logic.
- AWS Lake Formation - a service that makes it easy to set up a secure data lake in days, allowing you to ensure secure access to your sensitive data using granular controls at the column, row, and cell-levels. Lake Formation tag-based access control (LF-TBAC) works with IAM's attribute-based access control (ABAC) to provide fine-grained access to your data lake resources and data.
- AWS Glue - a serverless data integration service that makes it easy to discover, prepare, and combine data for analytics, machine learning, and application development. 
- AWS Lambda - a serverless, event-driven compute service that lets you run code for virtually any type of application or backend service without provisioning or managing servers.
- Amazon Redshift - uses SQL to analyze structured and semi-structured data across data warehouses, operational databases, and data lakes, using AWS-designed hardware and machine learning to deliver the best price performance at any scale.
- Amazon Athena - Amazon Athena is an interactive query service that makes it easy to analyze data in Amazon S3 using standard SQL. Athena is serverless, so there is no infrastructure to manage, and you pay only for the queries that you run

## Architecture

In this lab, we will create the following architecture.
![](/images/architecture.png)

## Before We Begin

Before you start, there are a few things you should know:
- The resources created in this lab may have some costs associated with them. 
- You will need an IAM user or role with appropriate permissions to execute this lab. This includes permissions to create the resources defined in the CloudFormation template, as well as the manual activities performed through the AWS Management Console. 
- It is assumed you're familiar with using and navigating through the AWS Management Console. 
- Changes to AWS Lake Formation will affect any existing resources that are using the data catalog. It is recommended that you complete this lab in a sandbox environment.

## Setup

In order to streamline this lab, a Cloud Formation template is used to create some initial resources. 

This template will create the following resources in your account:

- IAM Role - sforchlab-StepFunctionExecutionRole - this role will be used by the AWS Step Functions state machine. 
- IAM Role - sforchlab-GlueExecutionRole - this role will be used by the AWS Glue crawler.
- IAM Role - sforchlab-LambdaExecutionRole - this role will be used by the AWS Lambda function to load data.
- IAM Role - sforchlab-RedshiftExecutionRole - this role will be used by the AWS Lambda function to load data.
- S3 bucket - ${AWS::AccountId}-sforchlab-datalake - this bucket will hold the raw sales data.
- Glue Database - sales - database in the Glue catalog
- Glue Crawler - sforchlab-sales-raw-data-crawler - this crawler will crawl the raw data and create the raw_sales table in the catalog. 
- Glue Crawler - sforchlab-sales-processed-data-crawler - this crawler will crawl the processed data and create the processed_sales table in the catalog. 
- Glue Crawler - sforchlab-sales-aggregated-data-crawler - this crawler will crawl the processed data and create the aggregated_sales table in the catalog. 
- Glue ETL Job - sforchlab-sales-processing - this job will process the sales data into a more usable format and cleanse the data. 
- Lambda function - sforchlab-loaddataFunction - this function will load raw data into the S3 bucket.
- Step Function State Machine - sforchlab-datapipelineOrchestration-dataIngestion - an empty state machine to be built in this lab, to orchstrate the process of ingesting and processing the data to the data lake.
- Step Function State Machine - sforchlab-datapipelineOrchestration-aggregatedSales - an empty state machine to be built in this lab, to orchestrate the process of loading and processing data in Amazon Redshift.
- Step Function State Machine - sforchlab-datapipelineOrchestration-dataPipelineOrchestration - an empty state machine to be built in this lab, to orchestrate the process from end to end.
- VPCs and Subnet - to deply the Redshift cluster.
- Redshift cluster - sforchlab-warehouse-cluster - an empty state machine to be built in this lab, to orchestrate the process from end to end.

1. Click on the following link to launch the CloudFormation process. Please double check you are in the correct account and region.
[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=sforchlab-foundation&templateURL=https://s3.amazonaws.com/ee-assets-prod-us-east-1/modules/1e02deca779a4be58b9d50513a464cdc/v1/sforchlab/sforchlab-template.json)
2. Under parameters, enter a password for the Redshift cluster. This password must between 8 and 64 characters in length, contain at least one uppercase letter, one lowercase letter, and one number, and can use any ASCII characters with ASCII codes 33–126, except \' (single quotation mark), \" (double quotation mark), \\, \/, or \@.
3. For SubnetCIDR and VpcCIDR parameters values, default values have been suggested for use by the Redshift cluster instance. Please update these values as needed.
![](/images/cloudformation/stackdetails.PNG)
4. On the "Configure stack options" page, leave all the default values and click "Next".
5. On the final page, scroll to the end of the page, and mark the check box next to "I acknowledge..." and click "Create stack".
![](/images/cloudformation/ackcreate.PNG)
6. It should take roughly 5 minutes to complete.


## Step 1 - Configure AWS Lake Formation

:information_source: If you are already using AWS Lake Formation in this account, you can skip this step, and proceed directly to Step 2. 

:information_source: If you are not already using Lake Formation, but do have resources using the AWS Glue Catalog, please consider impact carefully and permissions will need to be updated. These steps should be completed in a non-production/sandbox account.

By default, AWS Lake Formation uses the IAMAllowedPrincipals group to provide super permissions to users and roles, effectively delegating access control to IAM policies. Before we can use AWS Lake Formation to manage access to our data, we need to revoke access provided by the IAMAllowedPrincipals group. In this way, IAM policies provide coarse grained permssions, while Lake Formation manages fine grained access control to catalog resources and data.
1. Navigate to [AWS Lake Formation](https://console.aws.amazon.com/lakeformation/) in the AWS Management console. Confirm that you are in the correct region.
2. The first time you use AWS Lake Formation, you need to configure an Adminstrator. On the "Welcome to Lake Formation" dialog, leave "Add myself" selected and click "Get started".
![](/images/lakeformation/welcome.PNG)
3. Click on "Settings, and uncheck the 2 checkboxes under "Data catalog settings", then click "Save".
![](/images/lakeformation/catalogsettings.PNG)
4. Click on "Administrative roles and tasks" in the navigation panel. 
5. Under "Database Creators", select the IAMAllowedPrincipals group, and click "Revoke".
6. Leave "Create database" checked, and click "Revoke".
![](/images/lakeformation/revokedbcreators.PNG)
7. Click on "Databases" in the navigation panel, and select the "sales" database.
8. Under the "Actions" dropdown, select "edit" for the database.
9. In the "Database details" page, uncheck the "Use only IAM access control..." checkbox, then click "Save".
![](/images/lakeformation/editdatabase.PNG)
10. Click on "Data lake permissions" in the navigation panel, and select the "IAMAllowedPrincipals" record for the sales database. Click "Revoke" and confirm that you want to revoke the permission.

## Step 2 - Grant Lake Formation permissions
Before we can process the data, we need to grant some permissions in AWS Lake Formation. For this lab, to keep it simple, we're going to grant permissions on all tables in the database. In real use case, it's best practice to use more granular permissions.

First, we'll grant permission for the Glue crawler to create tables in the Sales database.
1. Navigate to [AWS Lake Formation](https://console.aws.amazon.com/lakeformation/) in the AWS Management console. Confirm that you are in the correct Region.
2. Click on "Data lake permissions" in the navigation pane, and click "Grant".
3. Leave "IAM users and roles" selected, and choose the "sforchlab-GlueExecutionRole" in the "IAM users and roles" dropdown.
![](/images/lakeformation/grant-glue-principal.PNG)
3. Under "LF-Tags or catalog resources", select "Named data catalog resources".
4. In the "Databases" dropdown, select the "Sales" database. Leave the "Tables" dropdown empty.
![](/images/lakeformation/grant-glue-resources.PNG)
5. Under "Database permissions", mark the checkbox next to "Create table". Click "Grant".
![](/images/lakeformation/grant-glue-permissions.PNG)
Next, we'll grant permissions for Redshift to query the data.
6. Click on "Data lake permissions" in the navigation pane, and click "Grant".
7. Leave "IAM users and roles" selected, and choose the "sforchlab-RedshiftExecutionRole" in the "IAM users and roles" dropdown.
![](/images/lakeformation/grant-redshift-principal.PNG)
8. Under "LF-Tags or catalog resources", select "Named data catalog resources".
9. In the "Databases" dropdown, select the "Sales" database.
10. In the "Tables" dropdown, select "All tables"
![](/images/lakeformation/grant-redshift-resources.PNG)
11. Under "Database permissions", mark the checkboxes next to "Select" and "Describe" for "Table permissions" only. Click "Grant".
![](/images/lakeformation/grant-redshift-permissions.PNG)
Finally, we'll grant our user permission to query the table.
12. Click on "Data lake permissions" in the navigation pane, and click "Grant".
13. Leave "IAM users and roles" selected, and choose the user or role you're using to complete this lab.
![](/images/lakeformation/grant-user-principal.PNG)
14. Under "LF-Tags or catalog resources", select "Named data catalog resources".
15. In the "Databases" dropdown, select the "Sales" database.
16. In the "Tables" dropdown, select "All tables"
![](/images/lakeformation/grant-user-resources.PNG)
17. Under "Database permissions", mark the checkboxes next to "Select" and "Describe" for "Table permissions" only. Click "Grant".
![](/images/lakeformation/grant-user-permissions.PNG)

## Step 3 - Define Data Crawler State Machine
Next, we're going to create a state machine to run AWS crawlers.

:information_source: Note - for simplicity in this lab, we will be using a polling mechanism to confirm the completion of the crawler. In a real use case, you could instead use a Callback mechanism - use AWS EventBridge triggered by the change of state of the crawler to callback to the state machine to progress the flow. 

1. Navigate to [AWS Step Functions ](https://console.aws.amazon.com/stepfunctions/) in the AWS Management Console. Confirm that you are in the correct Region.
2. Click on "State machines" in the navigate panel. Click on the "sfochlab-RunGlueCrawler" state machine from the list.
3. Click the "Edit" button, then click the "Workflow Studio" button in the right hand corner.

:information_source: Much of the remainder of this lab will involve designing different state machines to orchestrate the data processinga and we will be using Workflow Studio to design the flow of states. The easiest way to do this is to use the "Search" box to find the state machine diagram. Clicking on each state will open a panel with the state configuration on the right hand side. 

4. Design your state machine as shown in the following diagram.
![](/images/stepfunctions/gluecrawler-1.PNG)

:information_source: To form the loop back from the "Wait" state to the "GetCrawler" state, click on the "Wait" state and select "GetCrawler" from the "Next state" dropdown.

5. Click on the "StartCrawler" state, and enter the following under "API Parameters". This will use the input to the state machine to specify the crawler to run.
```
{
  "Name.$": "$.Name"
}
```
![](/images/stepfunctions/gluecrawler-2.PNG)
6. On the Output tab of the "StartCrawler" activity, mark the checkbox next to "Transform result with ResultSelector...", and add the following into the text box:
```
{
  "Status": "Started"
}
```
7. Mark the checkbox next to "Add original input to output...". Leave "Combine original input with result" selected, and add the following into the textbox.
```
$.Status
```
![](/images/stepfunctions/gluecrawler-3.PNG)
8. Click on the GetCrawler state, and enter the following under "API Parameters":
```
{
  "Name.$": "$.Name"
}
```
![](/images/stepfunctions/gluecrawler-4.PNG)
9. On the "Output" tab, mark "Add original input to output...". Leave "Combine original input with result" selected, and enter the following into the textbox:
```
$.GetCrawler
```
![](/images/stepfunctions/gluecrawler-5.PNG)
10. Click on the "Rule #1" path from the "Choice" state, and click "Add conditions". Set the values as shown below:
- Variable = $.GetCrawler.Crawler.State
- Operator = is equal to
- Value = string constant / RUNNING
![](/images/stepfunctions/gluecrawler-6.PNG)
11. Your state machine should now be similar to the below:
![](/images/stepfunctions/gluecrawler-7.PNG)
12. Click "Apply and exit", then "Save". 
13. You will see a popup warning about IAM permissions. Click "Save anyway" to continue.
\
This state machine can now be used by other state machines to run a glue crawler and poll for the completion status. In the orchestration step function, we will pass in the name as the crawler to be run as an input.

## Step 4 - Define Redshift Orchestration State Machine

Next, we're going to create a state machine to run Amazon Redshift SQL statements.

1. Navigate to [AWS Step Functions ](https://console.aws.amazon.com/stepfunctions/) in the AWS Management Console. Confirm that you are in the correct Region.
2. Click on "State machines" in the navigate panel. Click on the "sfochlab-RedshiftQuery" state machine from the list.
3. Click the "Edit" button, then click the "Workflow Studio" button in the right hand corner.
4. Design your state machine as shown in the following diagram.
![](/images/stepfunctions/redshift-1.PNG)

:information_source: To form the loop back from the "Wait" state to the "GetCrawler" state, click on the "Wait" state and select "GetCrawler" from the "Next state" dropdown.

:information_source: You can click and drag the "Success" state from one path to the other.

5. Click on the "ExecutionStatement", and enter the following unider "API Parameters" - replace the value of <RedshiftClusterId> with your Redshift cluster ID from the output of the CloudFormation template.
```
{
  "ClusterIdentifier": "<RedshiftClusterId>",
  "Database": "dev",
  "DbUser": "admin",
  "Sql.$": "$.sql"
}
```
![](/images/stepfunctions/redshift-2.PNG)
6. Click on "DescribeStatement", and enter the following under "API Parameters".
```
{
  "Id.$": "$.Id"
}
```
![](/images/stepfunctions/redshift-3.PNG)
7. Click on the "Rule #1" path, then click "Add conditions". Add the following values:
- Variable = $.Status
- Operator = is equal to
- Value = string constant / FINISHED
![](/images/stepfunctions/redshift-4.PNG)
8. Click "Add new choice rule", then click "Add conditions" and enter the following values.
- Variable = $.Status
- Operator = is equal to
- Value = string constant / FAILED
![](/images/stepfunctions/redshift-5.PNG)
9. Drag a "Fail" state to the box labelled "Drop state here".
10. Click on the "Choice" activity again, ,and click "Add new choice rule", then click "Add conditions". Enter the following values.
- Variable = $.Status
- Operator = is equal to
- Value = string constant / ABORTED
![](/images/stepfunctions/redshift-6.PNG)
11. In the "ABORTED" choice rule, in the "Then next state is:" dropdown, select "Fail".
12. The final state machine should look similar to the following:
![](/images/stepfunctions/redshift-7.PNG)
13. Click "Apply and exit", then "Save". 
14. You will see a popup warning about IAM permissions. Click "Save anyway" to continue.
\
This state machine can now be used by other state machines to run a SQL statement using the Redshift Data API and poll for the completion status. In the orchestration step function, we will pass in the SQL statement as the crawler to be run as an input.


## Step 5 - Define Process Orchestration State Machine

Now that we've created our reusable state machines, let's create the process to orchestrate the full data pipeline.

1. Navigate to [AWS Step Functions ](https://console.aws.amazon.com/stepfunctions/) in the AWS Management Console. Confirm that you are in the correct Region.
2. Click on "State machines" in the navigate panel. Click on the "sforchlab-dataPipelineOrchestration" state machine from the list.
3. Click the "Edit" button, then click the "Workflow Studio" button in the right hand corner.
4. Design your state machine as shown in the following diagram.

:information_source: Note - you can right click on a state and select "Duplicate state".

![](/images/stepfunctions/orchn-1.PNG)

5. Click on the "Load Raw Sales Data" state. Under "API Parameters", select  "sforchlab-loaddataFunction$LATEST" as the "Function name"
![](/images/stepfunctions/orchn-2.PNG)
6. Click on the "Crawl Raw Sales" state, and enter the following under "API Parameters". Make sure you replace the values of "\<Region\>" and "\<AccountId\>".
```
{
  "StateMachineArn": "arn:aws:states:<Region>:<AccountId>:stateMachine:sforchlab-RunGlueCrawler",
  "Input": {
    "Name": "sforchlab-sales-raw-data-crawler",
    "AWS_STEP_FUNCTIONS_STARTED_BY_EXECUTION_ID.$": "$$.Execution.Id"
  }
}
```
7. Click on "Start Sales Transform Job", and add the following under "API Parameters". 
```
{
  "JobName": "sforchlab-sales-processing"
}
```
8. Make sure "Wait for task to complete" is selected.
![](/images/stepfunctions/orchn-3.PNG)
9. Click on the "Crawl Processed Sales" state, and enter the following under "API Parameters". Make sure you replace the values of "\<Region\>" and "\<AccountId\>".
```
{
  "StateMachineArn": "arn:aws:states:<Region>:<AccountId>:stateMachine:sforchlab-RunGlueCrawler",
  "Input": {
    "Name": "sforchlab-sales-processed-data-crawler",
    "AWS_STEP_FUNCTIONS_STARTED_BY_EXECUTION_ID.$": "$$.Execution.Id"
  }
}
```
10. Click on the "Redshift: Create Table" state, and enter the following under "API Parameters". Make sure you replace the values of "\<Region\>" and "\<AccountId\>".
```
{
  "StateMachineArn": "arn:aws:states:<Region>:<AccountId>:stateMachine:sforchlab-RedshiftQuery",
  "Input": {
    "sql": "DROP TABLE IF EXISTS sales CASCADE; CREATE TABLE sales (card_id bigint NOT NULL, customer_id bigint NOT NULL, price varchar(10), product_id bigint, timestamp timestamp) diststyle all;",
    "AWS_STEP_FUNCTIONS_STARTED_BY_EXECUTION_ID.$": "$$.Execution.Id"
  }
}
```
11. Click on the "Redshift: Load Data" state, and enter the following under "API Parameters". Make sure you replace all instances of the values of "\<Region\>" and "\<AccountId\>".
```
{
  "StateMachineArn": "arn:aws:states:<Region>:<AccountId>:stateMachine:sforchlab-RedshiftQuery",
  "Input": {
    "sql": "copy sales from 's3://<AccountId>-sforchlab-datalake/processed/sales/year=2021/month=12/' iam_role 'arn:aws:iam::<AccountId>:role/sforchlab/sforchlab-RedshiftExecutionRole' format as parquet;",
    "AWS_STEP_FUNCTIONS_STARTED_BY_EXECUTION_ID.$": "$$.Execution.Id"
  }
}
```
12. Click on the "Redshift: Create View" state, and enter the following under "API Parameters". Make sure you replace the values of "<Region>" and "<AccountId>".
```
{
  "StateMachineArn": "arn:aws:states:us-east-1:405685483016:stateMachine:sforchlab-RedshiftQuery",
  "Input": {
    "sql": "create view daily_sales_revenue as select sum(price) as daily_sales_revenue, trunc(timestamp) as day from sales where price !='' group by trunc(timestamp) order by trunc(timestamp);",
    "AWS_STEP_FUNCTIONS_STARTED_BY_EXECUTION_ID.$": "$$.Execution.Id"
  }
}
```
13. Click on the "Redshift: Unload" state, and enter the following under "API Parameters". Make sure you replace all instance of the values of "\<Region\>" and "\<AccountId\>".
```
{
  "StateMachineArn": "arn:aws:states:<Region>:<AccountId>:stateMachine:sforchlab-RedshiftQuery",
  "Input": {
    "sql": "unload ('select * from daily_sales_revenue') to 's3://<AccountId>-sforchlab-datalake/aggregated/daily_sales_revenue/' FORMAT AS parquet PARALLEL TRUE iam_role 'arn:aws:iam::<AccountId>:role/sforchlab/sforchlab-RedshiftExecutionRole';",
    "AWS_STEP_FUNCTIONS_STARTED_BY_EXECUTION_ID.$": "$$.Execution.Id"
  }
}
```
14. Click on the "Crawl Aggregated Sales" state, and enter the following under "API Parameters". Make sure you replace the values of "\<Region\>" and "\<AccountId\>".
```
{
  "StateMachineArn": "arn:aws:states:<Region>:<AccountId>:stateMachine:sforchlab-RunGlueCrawler",
  "Input": {
    "Name": "sforchlab-sales-aggregated-data-crawler",
    "AWS_STEP_FUNCTIONS_STARTED_BY_EXECUTION_ID.$": "$$.Execution.Id"
  }
}
```
15. Click "Apply and exit", then "Save". 
16. You will see a popup warning about IAM permissions. Click "Save anyway" to continue.

## Step 6 - Run the State Machine and Validate results

1. In Step Functions, click on the "sforchlab-dataPipelineOrchestration" state machine, then click "Start execution". 
2. In the popup, leave the default input, and click "Start execution".
3. This should may take up to 10 minutes to complete.
4. Navigate to [Amazon Athena](https://console.aws.amazon.com/athena) in the AWS Management Console. Confirm that you are in the correct Region.
5. If this is the first time you're using Amazon Athena, click the "Explore the query editor" button; otherwise, click on "Query editor" in the navigation panel.
6. Click on the "Settings" tab, then click on the "Manage" button.
7. In the "Manage settings" page, for "Query result location and encryption" browse to the bucket named "<accountID>-sforchlab-athena", then click "Save".
![](/images/athena/manage-settings.PNG)
8. Select the "Editor" tab, and execute the following statement:
```
SELECT * FROM "sales"."aggregated_daily_sales_revenue" limit 10;
```
![](/images/athena/aggregated_daily_sales_revenue.PNG)
9. You can also run queries against the raw_sales and processed_sales tables. 


## Clean Up

### Empty the S3 buckets
1. To delete the contents of the S3 buckets, navigate to [Amazon S3](https://console.aws.amazon.com/s3) in the AWS Management Console. 
2. For each bucket created in this lab, select the radio button next to the bucket, then click "Empty". 
3. Enter the text to confirm and delete the contents.
4. Complete these steps for both -datalake and the -athena buckets.

### Delete the CloudFormation stack
1. To delete the CloudFormation stack, navigate to the stack in the AWS Management Console, and click "Delete". 
2. This will remove all resources defined in the stack. Monitor the progress to ensure that all resources have been removed.
3. If there are any resources that fail to be deleted, try manually deleting those in the AWS Management Console and return to delete the CloudFormation stack. 

### Revert changes in AWS Lake Formation - optional.
1. To revert the Lake Formation changes, navigate to AWS Lake Formation in the AWS Management Console. 
2. Click on "Settings" in the navigation panel, then mark the two check boxes under "Default permissions for newly created databases and tables". Click Save.
3. Click on "Administrative roles and tasks" in the navigation panel. 
4. Under "Data lake administrators", click "Choose adminstrators". 
5. Remove your users/role by clicking the "X", then click "Save".
6. Under "Database creators, click Grant. Under "IAM users and roles", select the "IAMAllowedPrinciples" group. 
7. Select the checkbox next to "Create database" under "Catalog permissions", then click "Grant".
8. Under "Data lake permissions", select each row we created in this lab, and click "Revoke", then "Revoke" to confirm.
9. Under "LF-Tags" in the navigation panel, select the "Classification" tag we created earlier. Click "Delete", then enter the text to confirm. Click "Delete".