# Lab1 - Data Migration using AWS DMS
In this lab, we will be performing a migration of sample taxi related data from RDS Oracle to Amazon DynamoDB and Amazon Aurora PostgreSQL databases using AWS DMS.

## Prepare the Environment
First of all, check that the ClouFormation stack has successfully deployed all the AWS resources needed for this workshop. The `Outputs` section displays the output values of the stack.  

![stack-ok](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/c6ead0cc-7a19-489b-ac0e-4e297bf7e8ed)

Open a terminal window in the AWS Cloud9 IDE and run the following commands to clone the github repository.
```bash
cd ~/environment
git clone https://github.com/aws-samples/amazon-rds-purpose-built-workshop.git
```
Install PostgreSQL client and related libraries in the Cloud9 environment, so we can use the PostgreSQL command line utility **psql**.
```bash
wget https://yum.postgresql.org/12/redhat/rhel-6-x86_64/postgresql12-libs-12.13-1PGDG.rhel6.x86_64.rpm
wget https://yum.postgresql.org/12/redhat/rhel-6-x86_64/postgresql12-12.13-1PGDG.rhel6.x86_64.rpm

sudo yum clean all
sudo rpm -ivh postgresql12-libs-12.13-1PGDG.rhel6.x86_64.rpm
sudo rpm -ivh postgresql12-12.13-1PGDG.rhel6.x86_64.rpm
```
Install JQ  in the Cloud9 environment. It's a tool that we'll leverage to slice and filter JSON data.
```bash
sudo yum -y install jq gettext
```
Save the parent CloudFormation Stack name as a variable which will be used next to extract values for some of the output keys.
```bash
AWSDBWORKSHOP_CFSTACK_NAME=`aws cloudformation list-stacks | jq -r '.StackSummaries[].StackName' | grep "^Oracle"`
echo "export AWSDBWORKSHOP_CFSTACK_NAME=${AWSDBWORKSHOP_CFSTACK_NAME}" >> /home/ec2-user/.bashrc
. /home/ec2-user/.bashrc
echo $AWSDBWORKSHOP_CFSTACK_NAME
```
Set some required PostgreSQL environment variables by running the commands below in Cloud9 terminal to make it convenient to login to the database.
```bash
export DBENDP=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="AuroraClusterEndpointName") | .OutputValue')
echo $DBENDP
AURORADBMASTERUSER_NAME=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="AuroraDBMasterUser") | .OutputValue')
echo $AURORADBMASTERUSER_NAME
AURORADB_NAME=$(aws cloudformation describe-stacks --stack-name $AWSDBWORKSHOP_CFSTACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="AuroraDBName") | .OutputValue')
echo $AURORADB_NAME
echo "export PGHOST=$DBENDP" >> /home/ec2-user/.bashrc
echo "export PGPORT=5432" >> /home/ec2-user/.bashrc
echo "export PGUSER=$AURORADBMASTERUSER_NAME" >> /home/ec2-user/.bashrc
echo "export PGPASSWORD=auradmin123" >> /home/ec2-user/.bashrc
echo "export PGDATABASE=$AURORADB_NAME" >> /home/ec2-user/.bashrc
. /home/ec2-user/.bashrc
```
Now, we should be able to login to the Aurora PostgreSQL database just by running **psql** command.
```bash
psql
```
Run `\l` to list the databases in PostgreSQL cluster, `\dn` to list the schemas in the database and `\dt`to list the tables in the public schema of the database:

![sql-p](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/b6cec1fb-e310-44ed-b243-5c067b05773f)  
As you can see, there are no tables created in Aurora PostgreSQL yet.  

Before we migrate data from Oracle RDS to Aurora PostgreSQL, we need to setup a target schema. Execute the following commands in the Cloud9 terminal to create the schema:
```bash
cd ~/environment/amazon-rds-purpose-built-workshop/
psql -f ./src/create_taxi_schema.sql
```
Running `\dt` again after logging via psql shows that the tables were created.
![dt-psql](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/1e3b9ca7-12c1-4c93-b227-7ba9fee588d0)

## Creating Endpoints for Source and Target databases
Before we perform data migration using AWS DMS, we need to create endpoints for both source and target databases. It will be required for creating a migration task.
Create a source endpoint for Oracle RDS from the `Endpoints` section in the AWS DMS console and then test the connection to  the DMS replication instance created 
by the CloudFormation stack.
![test-ok](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/7732bfa5-fea5-4527-8a67-25574c7939d5)  
Now, create a Target endpoint for Aurora PostgreSQL and test the connection again.
![auraok](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/1c7d1865-7701-4dd0-a783-6f37c2fd1f8c)  

Lastly, create a Target endpoint for Amazon DynamoDB and make sure the connection status is successful after testing it.
![ddb ok](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/4801a835-b851-4a7f-8d46-ff6fa761d6a1)


## Migrate data from Oracle source to DynamoDB target
Using an AWS DMS task, we can specify which schema to migrate and the type of migration. In this lab, we will migrate existing data only and we will use map-record-to-record option 
to migrate trip data from Oracle to DynamoDB.  
Create a database migration task from the oracle source to the DynamoDB target. Enable CloudWatch logs and use the following mapping code:
```JSON
{  
"rules": [  
{  
    "rule-type": "selection",  
    "rule-id": "1",  
    "rule-name": "1",  
    "object-locator": {  
    "schema-name": "TAXI",  
    "table-name": "TRIPS"  
},  
    "rule-action": "include"  
},  
{  
    "rule-type": "object-mapping",  
    "rule-id": "2",  
    "rule-name": "2",  
    "rule-action": "map-record-to-record",  
    "object-locator": {  
    "schema-name": "TAXI",  
    "table-name": "TRIPS"  
},  
"target-table-name": "aws-db-workshop-trips",  
"mapping-parameters": {  
"partition-key-name": "riderid",  
"sort-key-name": "tripinfo",  
"attribute-mappings": [{  
    "target-attribute-name": "riderid",  
    "attribute-type": "scalar",  
    "attribute-sub-type": "string",  
    "value": "${RIDER_EMAIL}"  
},  
{  
    "target-attribute-name": "tripinfo",  
    "attribute-type": "scalar",  
    "attribute-sub-type": "string",  
    "value": "${PICKUP_DATETIME},${ID}"  
},  
{  
    "target-attribute-name": "driverid",  
    "attribute-type": "scalar",  
    "attribute-sub-type": "string",  
    "value": "${DRIVER_EMAIL}"  
},  
{  
    "target-attribute-name": "DriverDetails",  
    "attribute-type": "scalar",  
    "attribute-sub-type": "string",  
    "value": "{\"Name\":\"${DRIVER_NAME}\",\"Vehicle Details\":{\"id\":\"${VEHICLE_ID}\",\"type\":\"${CAB_TYPE_ID}\"}}"  
}]  
}  
}  
]  
}
```
Wait for the task status to change from Creating to Ready in the DMS console.
![ss2](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/61f42ddd-1984-44f8-8c15-6f1cd21ac855)  
We will modify a few DMS level task settings (`ParallelLoadThreads` and `ParallelLoadBufferSize`) to speed up the migration from Oracle source to DynamoDB target and then manually 
start the task.  
First, execute the following command in Cloud9 terminal to set the DMS task ARN in a variable:
```bash
TASK_ARN=$(aws dms describe-replication-tasks --filters Name=replication-task-id,Values=ora2ddb | jq -r '.ReplicationTasks[].ReplicationTaskArn')
```
Then, modify the DMS task settings by running the following in the Cloud9 terminal:
```bash
aws dms modify-replication-task --replication-task-arn $TASK_ARN --replication-task-settings '{"TargetMetadata":{"ParallelLoadThreads": 8,"ParallelLoadBufferSize": 50}}'
```
Start the task after it is ready again. You can review the replication task details in CloudWatch Logs. 
![watchlof](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/6bc8827d-899b-4874-9660-7e4517a7e83e)  
After the full load is complete, the Load state will change to Table completed and we will see that 128,714 rows are migrated.
![rowss](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/0003cc1e-ec28-4f13-937f-156d28d26acd)

## Migrate data from Oracle source to Aurora PostgreSQL target
Now, we will migrate four tables (Riders, Drivers, Payment and Billing) from RDS Oracle to Aurora PostgreSQL.  
Create a new migration task from the `orasource` to the `aurtarget` target. Enable CloudWatch logs, enable validation so the data for the tables will be compared between source and target after the full load is complete to make sure data was migrated accurately 
and use the following mapping code:
```JSON
{
"rules": [
  {
    "rule-type": "transformation",
    "rule-id": "1",
    "rule-name": "1",
    "rule-target": "schema",
    "object-locator": {
      "schema-name": "TAXI",
      "table-name": "%"
    },
    "rule-action": "convert-lowercase"
  },
  {
    "rule-type": "transformation",
    "rule-id": "2",
    "rule-name": "2",
    "rule-target": "table",
    "object-locator": {
      "schema-name": "TAXI",
      "table-name": "%"
    },
    "rule-action": "convert-lowercase"
  },
  {
    "rule-type": "transformation",
    "rule-id": "3",
    "rule-name": "3",
    "rule-target": "schema",
    "object-locator": {
      "schema-name": "TAXI"
    },
    "value": "public",
    "rule-action": "rename"
  },
  {
    "rule-type": "selection",
    "rule-id": "4",
    "rule-name": "4",
    "object-locator": {
      "schema-name": "TAXI",
      "table-name": "DRIVERS"
    },
    "rule-action": "include"
  },
  {
    "rule-type": "selection",
    "rule-id": "5",
    "rule-name": "5",
    "object-locator": {
      "schema-name": "TAXI",
      "table-name": "RIDERS"
    },
    "rule-action": "include"
  },
  {
    "rule-type": "selection",
    "rule-id": "6",
    "rule-name": "6",
    "object-locator": {
      "schema-name": "TAXI",
      "table-name": "BILLING"
    },
    "rule-action": "include"
  },
  {
    "rule-type": "selection",
    "rule-id": "7",
    "rule-name": "7",
    "object-locator": {
      "schema-name": "TAXI",
      "table-name": "PAYMENT"
    },
    "rule-action": "include"
  },
  {
    "rule-type": "transformation",
    "rule-id": "8",
    "rule-name": "8",
    "rule-target": "column",
    "object-locator": {
      "schema-name": "TAXI",
      "table-name": "%",
      "column-name": "%"
    },
    "rule-action": "convert-lowercase"
  }
]
}
```
Review the replication task details in CloudWatch Logs after the Load state for all the tables will change to `Table completed`.
![db ooPNG](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/335e6ddb-8dfe-4ead-85fc-a75fc43f1c39)


## Final Validation of DMS Tasks
Please check if both the DMS tasks are completed. You will see the below output.

- `ora2ddb` task status as Load Complete. Tables and Full load count should be: TRIPS (Count-128,714).
- `ora2aur` task status as Load Complete. Tables and Full load count should be: DRIVERS (Count-100,001), PAYMENT (Count-60,001), BILLING (Count -60,001), RIDERS (Count-100,000).

  ![succcc](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/69e44c5a-029a-4605-8b2e-44833e2961e6)






























