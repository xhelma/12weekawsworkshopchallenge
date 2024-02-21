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
