# Lab2 - Data processing using Amazon DynamoDB and Amazon Aurora
In this lab, we will simulate taxi trip booking by a rider and acceptance by a driver followed by billing and payment using Python scripts and SQL commands. 
We will utilize DynamoDB streams and AWS lambda functions to insert completed trip data from DynamoDB to Aurora PostgreSQL.  
The figure below depicts the high level deployment architecture.
<img width="1040" alt="architecture" src="https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/40c037b0-5bf2-4ca4-be33-f1961df366f6">

## Setup AWS Cloud 9 Environment
Open a new terminal window in the AWS Cloud9 IDE and update AWS SAM (Serverless Application Model) CLI to the latest version by running the following commands:
```bash
cd ~/environment
pip install --user --upgrade awscli aws-sam-cli
sam --version
```
Install Boto3  (AWS SDK for Python) by copying and pasting the following commands in the terminal:
```bash
cd ~/environment
curl -O https://bootstrap.pypa.io/pip/3.6/get-pip.py
sudo python3 get-pip.py
pip3 install boto3 --user
```
## Enable Amazon DynamoDB Streams
In this section, we will enable Amazon DynamoDB stream for the Amazon DynamoDB table named `aws-db-workshop-trips` that was created as part of the CloudFormation stack.  
Execuute the commands below in the Cloud9 terminal window to enable streams for the Amazon DynamoDB Tables named `aws-db-workshop-trips`:
```bash
STREAM_ID=$(aws dynamodb update-table --table-name aws-db-workshop-trips --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES | jq '.TableDescription.LatestStreamArn' | cut -d'/' -f4)

STREAM_NAME=stream/${STREAM_ID::-1}
echo "export AWSDBWORKSHOP_DDB_STREAM_NAME=${STREAM_NAME}" >> ~/.bash_profile
. ~/.bash_profile
echo $AWSDBWORKSHOP_DDB_STREAM_NAME
```
![stream](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/c983fa83-a846-453b-b952-6d3b26be49d6)

## Deploy AWS Lambda Function for DynamoDB Stream Integration

In this section, we will be using AWS Serverless Application Model (SAM ) CLI to deploy a Lambda Function within Amazon VPC. The SAM deployment will also include a Python interface to the 
PostgreSQL database engine as an AWS Lambda Layer.

This Lambda function will read the taxi trip information from DynamoDB streams as they are inserted/updated in the DynamoDB table `aws-db-workshop-trips`. Only when a trip is complete 
(denoted by the STATUS attribute in the trip item/record), the Lambda function will insert information into trips table in Aurora PostgreSQL.  
First, we will download and package the binaries for `PG8000`  - a Python interface to PostgreSQL. The package will be deployed as an AWS Lambda Layer. 
Copy and paste the below commands in the Cloud9 terminal window.
```bash
cd ~/environment
mkdir pglayer
virtualenv -p python3 pglayer
cd pglayer
source bin/activate
mkdir -p pg8000-layer/python
pip install pg8000 -t pg8000-layer/python
cd pg8000-layer
zip -r pg8000-layer.zip python
mkdir ~/environment/amazon-rds-purpose-built-workshop/src/ddb-stream-processor/dependencies/
cp ~/environment/pglayer/pg8000-layer/pg8000-layer.zip ~/environment/amazon-rds-purpose-built-workshop/src/ddb-stream-processor/dependencies/
```
Then, we will deploy a SAM template that contains the configuration for the Lambda function and the Lambda Layer.  
To validate the SAM template, copy and paste the commands below in the Cloud9 terminal window.
```bash
cd ~/environment/amazon-rds-purpose-built-workshop/src/ddb-stream-processor
sam validate
```
![pp2](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/563af03d-f8ac-4c7b-b3b3-5b5d5c760723)  
To package the AWS SAM application, copy and paste the commands below in the Cloud9 terminal window. This will create a file named `template-out.yaml` in the same folder 
and will upload the packaged binaries to the specified Amazon S3 bucket.
```bash
cd ~/environment/amazon-rds-purpose-built-workshop/src/ddb-stream-processor
S3_BUCKETNAME=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="S3bucketName") | .OutputValue')
echo "export S3_BUCKETNAME=${S3_BUCKETNAME}" >> /home/ec2-user/.bashrc
. /home/ec2-user/.bashrc
echo $S3_BUCKETNAME
sam package --output-template-file template-out.yaml --s3-bucket $S3_BUCKETNAME
```
Ensure that the packages have been uploaded successfully to the Amazon S3 bucket by entering the following command:
```bash
aws s3 ls s3://$S3_BUCKETNAME
```
Set some environment variables from the output of the Amazon CloudFormation stack that was deployed. We will use these environment variables while deploying the Lambda function in the next
step.
```bash
AURORADB_NAME=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="AuroraDBName") | .OutputValue')
AURORACLUSTERENDPOINT_NAME=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="AuroraClusterEndpointName") | .OutputValue')
AURORADBMASTERUSER_NAME=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="AuroraDBMasterUser") | .OutputValue')
LAMBDASECURITYGROUP_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSecurityGroupId") | .OutputValue')
LAMBDASUBNET1_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSubnet1") | .OutputValue')
LAMBDASUBNET2_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSubnet2") | .OutputValue')
echo $AURORADB_NAME
echo $AURORACLUSTERENDPOINT_NAME
echo $AURORADBMASTERUSER_NAME
echo $LAMBDASECURITYGROUP_ID
echo $LAMBDASUBNET1_ID,$LAMBDASUBNET2_ID
```
![vall](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/4e76debd-d944-4e3c-be90-737976396dea)  
Copy and paste the following command to deploy the Lambda Function along with the Lambda Layer:
```bash
cd ~/environment/amazon-rds-purpose-built-workshop/src/ddb-stream-processor
sam deploy --template-file template-out.yaml --capabilities CAPABILITY_IAM --stack-name SAM-DDB-STREAM-APG --parameter-overrides LambdaLayerNameParameter=aws-db-workshop-pg8000-layer DDBStreamName=$AWSDBWORKSHOP_DDB_STREAM_NAME SecurityGroupIds=$LAMBDASECURITYGROUP_ID VpcSubnetIds=$LAMBDASUBNET1_ID,$LAMBDASUBNET2_ID DatabaseName=$AURORADB_NAME DatabaseHostName=$AURORACLUSTERENDPOINT_NAME DatabaseUserName=$AURORADBMASTERUSER_NAME DatabasePassword=auradmin123
```
Finally, a Lambda function named `aws-db-workshop-ddb-stream-processor` deployed is to be seen on the AWS Lambda console navigation bar.
![lambda](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/5de5b474-43d6-4410-a184-c4109d04128b)

## Deploy AWS Lambda Functions for Taxi Ride workflow
In this section, we will be using AWS Serverless Application Model (SAM ) CLI to deploy Lambda Functions within Amazon VPC which will be used to simulate Taxi Ride workflow.
First, In this step, we will deploy a SAM template that contains configuration for three Lambda functions and a Lambda Layer. The lambda layer contains taxi ride workflow utility functions written in Python. 
The three Lambda functions are used for booking a taxi trip by a rider, accepting a trip by a driver and then completing a trip by a driver.
To validate the SAM template, execute the commands below in the Cloud9 terminal window.
```bash
cd ~/environment/amazon-rds-purpose-built-workshop/src/taxi-ride-workflow
sam validate
```
To package the AWS SAM application, copy and paste the commands below in the Cloud9 terminal window. This will create a file named template-out.yaml in the same folder and will 
upload the packaged binaries to the specified Amazon S3 bucket.
Set some environment variables from the output of the Amazon CloudFormation stack that was deployed. 
```bash
cd ~/environment/amazon-rds-purpose-built-workshop/src/taxi-ride-workflow
echo $S3_BUCKETNAME
sam package --output-template-file template-out.yaml --s3-bucket $S3_BUCKETNAME --region us-east-1
```
![ss255](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/d2e5e642-e800-40d0-a8f4-92c0de107738)
Set some environment variables from the output of the Amazon CloudFormation stack that was deployed.
```bash
LAMBDASECURITYGROUP_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSecurityGroupId") | .OutputValue')
LAMBDASUBNET1_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSubnet1") | .OutputValue')
LAMBDASUBNET2_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSubnet2") | .OutputValue')
echo $LAMBDASECURITYGROUP_ID
echo $LAMBDASUBNET1_ID,$LAMBDASUBNET2_ID
```
Copy and paste the following command to deploy the Lambda Functions along with the Lambda Layer:
```bash
LAMBDASECURITYGROUP_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSecurityGroupId") | .OutputValue')
LAMBDASUBNET1_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSubnet1") | .OutputValue')
LAMBDASUBNET2_ID=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="LambdaSubnet2") | .OutputValue')
echo $LAMBDASECURITYGROUP_ID
echo $LAMBDASUBNET1_ID,$LAMBDASUBNET2_ID
```
We should see now three Lambda functions named rider-book-trip, driver-accept-trip and driver-complete-trip deployed when we click Functions  on the AWS Lambda console navigation bar.
![3func](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/daf357fc-b782-4c97-9953-38c2cbfbe22c)

## Create and Deploy API for Taxi Ride workflow

In this section, we will create and deploy API using Amazon API Gateway which will be used to simulate Taxi Ride workflow. API gateway allows you to define Resources and Methods. We will create three resources used for booking a taxi trip by a rider, accepting a trip by a driver and completing a trip by a driver. Then you will create the relevant HTTP method (**GET** in our case) for 
each resource which will be used to front the REST API's. Finally, you will publish the API.  
Create a regional REST API named `taxi_trip_apis`. Then create three resources called `riderbook`, `driveraccept` and `drivercomplete` respectively.  
Create a GET method for the `/driveraccept` resource using the `driver-accept-trip` Lambda function. Do the same thing for `/drivercomplete` resource using the `driver-complete-trip` Lambda 
function and the `/riderbook` using the `rider-book-trip` Lambda function.
![3ddd](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/b046a27f-4325-482a-8184-7178b8c4f92b)  
Now you can deploy the API by choosing a new stage called `test`.
## Taxi Ride Workflow
In this section, we will simulate booking of a taxi trip by a rider followed by acceptance and completion of the trip by a driver using Amazon API Gateway. 
After the trip is complete, we will run a PL/pgSQL procedure in Aurora PostgreSQL to process billing and driver payments.  
From the `Stages`, choose the GET method for `/riderbook` resource and copy the Invoke URL shown at the top. We will use this URL to invoke the rider-book-trip AWS Lambda function.
We need to pass `rider_id` and `rider_mobile` parameters to the `rider-book-trip` Lambda function for booking a taxi trip. As an example, we will book a trip for the `rider_id` 71463 
with `rider_mobile` +11492133668.  Append the URL with these two parameters and their corresponding values as follows.
```
<riderbook GET URL>?rider_id=71463&rider_mobile=%2B11492133668
```
Browsing to the URL, we get the following output:
![lamn1](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/a6ad911a-5d34-4158-91e5-6cbbd6ec366b)  
Choose the GET method for `/driveraccept` resource and copy the Invoke URL. We will use this URL to invoke the `driver-accept-trip` AWS Lambda function. We need to pass `rider_id` and
`trip_info` parameters to the `driver-accept-trip` Lambda function for accepting a taxi trip by a driver. 
Append the same `rider_id` you used above for booking the trip and pass the value for `trip_info` that we got in the previous output.
![tt6](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/1b19b755-0db5-42df-8702-e04439b8e561)  
Choose the GET method for `/drivercomplete` resource and copy the Invoke URL. We need to pass `rider_id` and `trip_info` parameters to the `driver-complete-trip` Lambda function for 
completing a taxi trip by a driver. Append the same `rider_id` we used above for booking the trip and pass the value for `trip_info`.
![fini](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/9d6dd8ee-7681-4336-87f1-ac9494afa962)  
Copy and paste the commands below in the Cloud9 terminal window to connect to the Aurora PostgreSQL database and review the details for the trip we just completed. 
Notice that the status of the trip is marked as Completed.
```bash
psql
\x
select * from trips;
```
Copy and paste the command below in the same psql session. This will execute the billingandpayments PL/pgSQL procedure to simulate billing and payment workflow.
```bash
call billingandpayments();
```
![inn](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/2f88ba60-bba1-4cfd-9b4c-b49715fd963a)
























