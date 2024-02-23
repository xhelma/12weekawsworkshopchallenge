# Overview
In this workshop, we are going to explore several services that make part of the AWS analytics portfolio such as AWS Glue, Amazon Athena, Amazon Kinesis, Amazon EMR and Amazon 
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
```json
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










