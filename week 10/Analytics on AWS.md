# Overview
In this workshop[^1], we are going to explore several services that make part of the AWS analytics portfolio such as AWS Glue, Amazon Athena, Amazon Kinesis, Amazon EMR and Amazon 
Quicksight. The following serveless data lake architecture is designed in order to ingest, store, transform and analyze the data. 

<img width="989" alt="lab-architecture" src="https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/c14f4f81-48b4-47b7-b10c-7aa986b39425">

# Ingest and Store
We will start by creating a S3 bucket in the `us-east-1` region that is going to host the raw and reference data. Then, we are going to add a `reference_data` folder nested inside a `data` folder and upload the `tracks_list.json` file to it. 

![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/17998014-c33f-4eb3-a668-f31b6002d9c6)

Next, we'll create a Kinesis Firehose delivery stream to ingest the data and store it inside the S3 bucket. It has the name of `analytics-workshop-stream` with a source of a `Direct PUT` and a destination of the S3 bucket that we have previously created. We won't configure any transformations of the data. We specify the bucket prefix to be `data/raw/`.
We'll set the bucket size to `1 MiB` with an interval of `60 seconds`. No encryption or compression is enabled.

![FIREHOSE](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/fbc65ef3-b9ff-463c-a422-03af743e909b)

Finally, we'll set up the Kinesis Data Generator to produce fake data that will be the source of our raw data ingested into Kinesis Firehose. To do so, we will launch a CloudFormation stack using https://aws-kdg-tools-us-east-1.s3.amazonaws.com/cognito-setup.json as template. When completed, navigate to the Kinesis Data Generator tool URL and log into it. Pick the `us-east-1` region and specify `analytics-workshop-stream` as the delivery stream and set the records per second to `2000`.
This is the record template that will be used:
```js
{
    "uuid": "{{random.uuid}}",
    "device_ts": "{{date.utc("YYYY-MM-DD HH:mm:ss.SSS")}}",
    "device_id": {{random.number(50)}},
    "device_temp": {{random.weightedArrayElement(
    {"weights":[0.30, 0.30, 0.20, 0.20],"data":[32, 34, 28, 40]}
    )}},
    "track_id": {{random.number(30)}},  
    "activity_type": {{random.weightedArrayElement(
        {
            "weights": [0.1, 0.2, 0.2, 0.3, 0.2],
            "data": ["\"Running\"", "\"Working\"", "\"Walking\"", "\"Traveling\"", "\"Sitting\""]
        }
    )}}
}
```
We'll send around 10,000 messages to Kinesis before stopping it. The data will populate the S3 bucket using yyyy/mm/dd/hh partitioning.

# Catalog Data
Now that we have the data, we can register it in the AWS Data Catalog before being able to query it.

We need to give permissions to the AWS Glue service so that it can access the data stored in S3 and create the necessary entities in the Glue Data Catalog. Hence, we'll create an IAM role named `AnalyticsworkshopGlueRole` that has the two policies `AmazonS3FullAccess` and `AWSGlueServiceRole` attached to it.
Next, we'll create an AWS Glue crawler with the name of `AnalyticsworkshopCrawler` that will discover the schema of the data. Then, we add the S3 bucket as our data source. We attach to it the previous IAM role and create a database named `analyticsworkshopdb` that will be the target. Set the crawler schedule frequency to be on demand. Once created, run it. Checking the database target, we can see the schema of our dataset.
 
 ![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/1cc6961d-86a8-41cd-a4eb-407eb5e899d3)

It's time to query the data using Amazon Athena. Edit the setting of the editor by specifying our S3 bucket as the location of the query result, specifically under the `query_results` folder. 
In the query editor, we run the following query:
```sql
SELECT activity_type,
         count(activity_type)
FROM raw
GROUP BY  activity_type
ORDER BY  activity_type
```
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/eadab447-375d-4a03-a17b-978b78b656b5)

# Transform Data with AWS Glue
## Using Interactive Sessions
We'll transform our data using AWS Glue Interactive Sessions with the aid of Glue Studio and Jupyter Notebooks.
We start by creating an IAM policy with the name `AWSGlueInteractiveSessionPassRolePolicy` and the following policy statement:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
    "Effect": "Allow",
    "Action": "iam:PassRole",
    "Resource":"arn:aws:iam::<AWS account ID>:role/Analyticsworkshop-GlueISRole"
    }
  ]
}
```
We proceed to create the `Analyticsworkshop-GlueISRole` IAM role that is used in the policy statement. We attach to it 4 policies: `AWSGlueServiceRole`, `AwsGlueSessionUserRestrictedNotebookPolicy`, `AWSGlueInteractiveSessionPassRolePolicy` and `AmazonS3FullAccess`.

Now, we can create an AWS Glue job with a PySpark Jupyter Notebook using the notebook file provided in the workshop. We name it `AnalyticsOnAWS-GlueIS` and run the code blocks one cell at time. The goal is to create a dynamic data frames for each of our raw and reference data, join them based on the `track_id` column, clean it by removing unnecessary columns using the `DropFields` transform and finally store it to our S3 bucket in Parquet format which is optimized for fast retrievel of data.
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/e5e90821-f7f2-41c9-bf1c-4269aa12712e)


## Using AWS Glue Studio
In this step, we will do the same ETL process as before but with the graphical interface. 

From the AWS Glue studio visual UI, we add both the raw and reference data catalog tables stored in the S3 bucket as source then we add a transform to the reference data one to change the `track_id` into an `int` type. Then join it with the raw data with the condition being joined by the `track_id` column. For the joined result, we are going to drop any column relating to the partition and change the `track_id` again to a `string` data type. Finally, we"ll add a S3 bucket as a target that is going to host the processed data in the parquet format. We save the job under the name `AnalyticsOnAWS-GlueStudio` and run it. This will also generate a script that we can edit and reuse for future needs.
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/522ad883-aa77-44a3-8f4f-02e6872ee3bf)

# Transform Data with AWS Glue DataBrew

We'll try the same ETL process but with a new AWS service, which is AWS Glue Databrew.

We start by creating an AWS Glue Databrew project named `AnalyticsOnAWS-GlueDataBrew` and import the raw data table into it. We create the IAM role `AnalyticsOnAWS-GlueDataBrew` directly the project creation's console.
From the grid view, we change the `track_id` data type to a `string` type. Then, we run a data profile to uncover informative statistics of our data. While waiting for it to finish, we'll join the reference data table after adding it with the key `track_id`. We create and run the job that we name `AnalyticsOnAWS-GlueDataBrew-Job` and output it in the S3 bucket in `Glueparquet` format.
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/6f61ace3-f11d-430b-9416-6df886ee7ed9)

# Transform Data with EMR

 In this step we are going to use Amazon EMR to submit pyspark jobs to read and transform the raw data.

First, we are going to upload `emr_pyspark.py` script file to S3 bucket under the folder `scripts` and create another folder named `logs` that is going to host the EMR logs.
Then, we are going to create an EMR cluster of the name of `analytics-workshop-transformer` that uses the Spark application bundle and add a step that submits work to the application framework on the cluster. In case of failure, we set the cluster to terminate. We also specify the logs folder we previously created to house the cluster logs that are to be generated. We are going to let the EMR to create an EC2 instance profile that will assign a role to every EC2 instances that is part of the cluster. Moreover, we will grant it read and write access to all the S3 buckets of our account.
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/42532d7b-852d-4970-a681-412211b0f536)

Finally, we're going to rerun the `AnalyticsworkshopCrawler` crawler. After comptetion, we can confirm from the database section that the `emr_processed_data` table was added.


# Analyze with Athena
Now that we have the data generated, stored and cataloged, we can analyze by runing simple SQL queries using Amazon Athena.
From the query editor, select the `analyticsworkshopdb` as a source and run the following query:
```sql
SELECT artist_name,
       count(artist_name) AS count
FROM processed_data
GROUP BY artist_name
ORDER BY count desc
```
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/0d08b579-640d-4914-9c60-29cf8ca6acd7)

# Analyze with Kinesis Data Analytics
This time, we will perform real-time analysis of streaming data using Amazon Kinesis Data Analytics. 

Firstly, we'll create an IAM role with the name of `AnalyticsworkshopKinesisAnalyticsRole` that has the two policies `AWSGlueServiceRole` and `AmazonKinesisFullAccess` attached to it. Next, we'll create a Kinesis data stream called `analytics-workshop-data-stream` in capacity mode with two shards. Then, we'll add a new table to the  `analyticsworkshopdb` AWS Glue database with the name of `raw_stream` and specify the previously created data stream as source. We add the following schema in json text:
```json
[
  {
    "Name": "uuid",
    "Type": "string",
    "Comment": ""
  },
  {
    "Name": "device_ts",
    "Type": "timestamp",
    "Comment": ""
  },
  {
    "Name": "device_id",
    "Type": "int",
    "Comment": ""
  },
  {
    "Name": "device_temp",
    "Type": "int",
    "Comment": ""
  },
  {
    "Name": "track_id",
    "Type": "int",
    "Comment": ""
  },
  {
    "Name": "activity_type",
    "Type": "string",
    "Comment": ""
  }
]
```
Now, we can create the Kinesis Analytics Streaming Application Studio Notebook, which processes streaming data from the Kinesis Data Stream and allow us to write SQL analytical queries to get real-time insights. Let's name it `AnalyticsWorkshop-KDANotebook` with the runtime option `Apache Flink 1.11, Apache Zeppelin 0.9`. it will assume the `AnalyticsworkshopKinesisAnalyticsRole` IAM role and the metadata will be defined by the `analyticsworkshopdb` AWS Glue database.
After running and opening the Zeppelin Notebook, we create a note named `AnalyticsWorkshop-ZeppelinNote`  and we insert the following SQL query to list down all streaming data from our Kinesis Data Generator: 
```sql
%flink.ssql(type=update)

SELECT * FROM raw_stream;
```
Add another paragraph and insert this SQL query that counts the number of activities currently being done:
```sql
%flink.ssql(type=update)

SELECT activity_type, count(*) as activity_cnt FROM raw_stream group by activity_type;
```
So far no data is being sent from the Kinesis Data Generator. To change that, we access its URL again and log in. We choose the `analytics-workshop-data-stream` stream and set
the records per second rate to 100 and configure the record to have the following template:
```js
{
  "uuid": "{{random.uuid}}",
  "device_ts": "{{date.utc("YYYY-MM-DD HH:mm:ss.SSS")}}",
  "device_id": {{random.number(50)}},
  "device_temp": {{random.weightedArrayElement(
    {"weights":[0.30, 0.30, 0.20, 0.20],"data":[32, 34, 28, 40]}
  )}},
  "track_id": {{random.number(30)}},  
  "activity_type": {{random.weightedArrayElement(
        {
            "weights": [0.1, 0.2, 0.2, 0.3, 0.2],
            "data": ["\"Running\"", "\"Working\"", "\"Walking\"", "\"Traveling\"", "\"Sitting\""]
        }
    )}}
}
```
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/c2c34f8f-77a4-43e4-b2dc-6172d02c07e0)

# Visualize in Quicksight
We are going to use the Quicksight AWS service to visualize our processed data.

We sign up for the entreprise edition of Quicksight and we allow access to Amazon Athena and our S3 bucket. From the dashboard, we create a new Athena dataset with the name of `analyticsworkshop`. We choose the `processed_data` table from the `analyticsworkshopdb` database.

### Visualization 1: Heat map of users and tracks they are listening to
Let's create a heat map. We set the row parameter to `device_id` and the column parameter to `track_name`.
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/2f3d3537-59b5-42c4-b6f6-a2ea2dcbf590)

### Visualization 2: Tree map of most played Artist Names
We'll add another visual of type Tree Map. We pick `artist_name` as a field list.
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/d89c2ea9-d5e5-4614-8dba-d0d32f7bcd46)

# Serve with Lambda
In this section, we are going to create a Lambda Function that will host the code for Athena to query and fetch the top 5 popular songs by hits from processed data in S3.
We name the Lambda function `Analyticsworkshop_top5Songs`, set the runtime to 3.8 and insert the following code:
```py
import boto3
import time
import os

# Environment Variables
DATABASE = os.environ['DATABASE']
TABLE = os.environ['TABLE']
# Top X Constant
TOPX = 5
# S3 Constant
S3_OUTPUT = f's3://{os.environ["BUCKET_NAME"]}/query_results/'
# Number of Retries
RETRY_COUNT = 10

def lambda_handler(event, context):
    client = boto3.client('athena')
    # query variable with two environment variables and a constant
    query = f"""
        SELECT track_name as \"Track Name\", 
                artist_name as \"Artist Name\",
                count(1) as \"Hits\" 
        FROM {DATABASE}.{TABLE} 
        GROUP BY 1,2 
        ORDER BY 3 DESC
        LIMIT {TOPX};
    """
    response = client.start_query_execution(
        QueryString=query,
        QueryExecutionContext={ 'Database': DATABASE },
        ResultConfiguration={'OutputLocation': S3_OUTPUT}
    )
    query_execution_id = response['QueryExecutionId']
    # Get Execution Status
    for i in range(0, RETRY_COUNT):
        # Get Query Execution
        query_status = client.get_query_execution(
            QueryExecutionId=query_execution_id
        )
        exec_status = query_status['QueryExecution']['Status']['State']
        if exec_status == 'SUCCEEDED':
            print(f'Status: {exec_status}')
            break
        elif exec_status == 'FAILED':
            raise Exception(f'STATUS: {exec_status}')
        else:
            print(f'STATUS: {exec_status}')
            time.sleep(i)
    else:
        client.stop_query_execution(QueryExecutionId=query_execution_id)
        raise Exception('TIME OVER')
    # Get Query Results
    result = client.get_query_results(QueryExecutionId=query_execution_id)
    print(result['ResultSet']['Rows'])
    # Function can return results to your application or service
    # return result['ResultSet']['Rows']
```
We set the timeout to 10 and we add three environment variables:
- Key: `DATABASE`, Value: `analyticsworkshopdb`
- Key: `TABLE, Value: `processed_data`
- Key: `BUCKET_NAME`, Value: `yourname-analytics-workshop-bucket`
We edit its IAM role by attaching to it the two policies `AmazonS3FullAccess` and `AmazonAthenaFullAccess` and we deploy the function so we can test it with a simple hello world test.
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/e54ab7d9-c7f3-4c33-8c90-513087a7583b)
From the Athena editor, we run the following query:
```sql
SELECT track_name as "Track Name",
    artist_name as "Artist Name",
    count(1) as "Hits" 
FROM analyticsworkshopdb.processed_data 
GROUP BY 1,2 
ORDER BY 3 DESC 
LIMIT 5;
```
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/68f02ccf-b8e6-4f0e-933b-ba2742aef815)
We notice that the query result is identical to the lamdba function output.  

# Warehouse on Redshift
We are going to setup an Amazon Redshift cluster that the data will be loaded into it using AWS Glue. 

We first start by creating an IAM role for the Redshift cluster that we name `Analyticsworkshop_RedshiftRole` and attach the policies `AmazonS3FullAccess` and `AWSGlueConsoleFullAccess` to it.
Next, we create a Redshift cluster with two nodes of type ` dc2.large`. We associate to it the previously created IAM role.
We are also going to create an S3 Gateway Endpoint so that Redshift cluster can communicate with S3 over the private network. For the service, we pick the `com.amazonaws.us-east-1.s3` gateway endpoint. We are going to edit the security group of the redshift cluster to include an inbound rule that allows HTTPS traffic from the S3 gateway and an outbound rule that allows all traffic while referecing the same security group.

After that, we will create a Redshift Connection under Glue connection named `analytics_workshop` which we can use in development endpoint to establish a connection to Redshift. From the Redshift query editor, we establish a new connection to the redshift database then we execute the query below to create schema and tables for raw and reference data:
```sql
--    Create redshift_lab schema.
CREATE schema redshift_lab;
--    Create f_raw_1 table.
CREATE TABLE IF not EXISTS redshift_lab.f_raw_1 (
  uuid            varchar(256),
  device_ts         timestamp,
  device_id        int,
  device_temp        int,
  track_id        int,
  activity_type        varchar(128),
  load_time        int
);
--    Create d_ref_data_1 table.
CREATE TABLE IF NOT EXISTS redshift_lab.d_ref_data_1 (
  track_id        int,
  track_name    varchar(128),
  artist_name    varchar(128)
);
```
Now, we'll create an ETL Glue job called `AnalyticsOnAWS-Redshift`using the Jupyter notebook provided in the workshop and run the code cell by cell.
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/7ec34f77-a8cf-424b-8cca-69597dd2ca40)

Once the ETL script has ran successfully, we validate that the data has arrived in Redshift by running the following query that checks  the number of records in raw and reference data tables from the Redshift Query Editor:
```sql
select count(1) from redshift_lab.f_raw_1;

select count(1) from redshift_lab.d_ref_data_1;
```
![image](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/20ef8acc-e265-4a0e-ba1c-23efd90e47db)


[^1]: [Analytics on AWS](https://catalog.us-east-1.prod.workshops.aws/workshops/44c91c21-a6a4-4b56-bd95-56bd443aa449/en-US)


