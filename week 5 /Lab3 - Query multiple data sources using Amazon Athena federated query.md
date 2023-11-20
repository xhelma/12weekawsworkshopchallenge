# Lab3 - Query multiple data sources using Amazon Athena federated query
In this lab, we will leverage Athena federated query feature to query both DynamoDB and Aurora trip data using a single SQL query. 
We will also utilize this feature to query DynamoDB and data stored in Amazon S3.  
The figure below depicts the high level deployment architecture.
![lab3-arch](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/12a313c2-779b-4b26-acda-0953dafd74ea)

## Prepare the Environment
Connect to the Aurora PostgreSQL cluster using the below command. Enter `auradmin123` as password when prompted.
```bash
psql -h $AURORACLUSTERENDPOINT_NAME -U $AURORADBMASTERUSER_NAME -d $AURORADB_NAME
```
Run the below SQL Command to check the trips table Aurora PostgreSQL. We will be using this table for federared query using Athena.
```SQL
select * from trips;
select rider_email, trip_info from trips;
```
Launch a new query editor from the Amazon Athena service console and edit the primary workgroup to add the S3 bucket deployed using the CloudFormation stack, select the version 3 Athena engine 
and enable publishing query metrics to AWS CloudWatch.
![eee](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/0f2d5846-e6fd-447e-af0d-66e596af978f)

## Setup Athena Connectors and Catalogs
In this section, we will first deploy a connector for DynamoDB data source and then a connector for Aurora PostgreSQL data source.
The Amazon DynamoDB Data Source Connector enables Amazon Athena to communicate with DynamoDB, making the trips tables accessible via SQL.  
Create a new DynamoDB data source named `ddbcatalog` and create a new Lambda function that we will call `taxiddb` and we'll use the S3 bucket already created by the CloudFormation stack
as SpillBucket.
We should see now `taxiddb` Lambda function deployed. 
![jbb](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/3f2cb31c-8700-4f25-ab8b-e36c73626a2b)  
Go back to the previous Athena Connect data sources window and under the Lambda functions section, choose the `taxiddb` one.  
Now, we need to repeat the same process to deploy connector for Aurora PostgreSQL data source. This connector enables Amazon Athena to access your Amazon RDS and Amazon Aurora databases using JDBC driver. 
Create a new PostgreSQL data source named `rdbcatalog` and create a new Lambda function that we will call `taxirdb` and we'll use the S3 bucket, the security group and the two subnets already created by the CloudFormation 
stack.  
We should see now the `taxirdb` Lambda function deployed.
![nnnn](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/ff5a9004-1ae9-46ea-9992-504458cb5358)
Add an environment variable to this Lambda function with the key `rdbcatalog_connection_string` and value same as the value of the `default` key. This is required for Athena to connect to multiple database instances of any 
type using a single Lambda function.  
Now, we are ready to query both DynamoDB and Aurora PostgreSQL using Athena federated query.

## Query multiple data sources using Athena Federated Query
In Lab 2, we stored taxi trip data in DynamoDB and replicated completed trip records to Aurora PostgreSQL using DynamoDB streams and AWS lambda function. Now, we will validate data accuracy and 
consistency of the trip record between Amazon DynamoDB and Amazon Aurora using Athena Federated query.  

Run some sample queries on the Aurora PostgreSQL database using the catalog name from the query editor:
- Query to list the databases in `rdbcatalog` catalog:
![222PNG](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/cfcaf497-4936-4a65-ab24-f1c241b6bf4c)
- Query to list the tables in public schema of `rdbcatalog` catalog:
![33](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/84755876-2756-46db-bfd7-f882db8c1291)
- Query to view the trip records in `rdbcatalog` catalog:
![455](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/dc5340e5-269e-4a5a-aa77-22f04c71b045)
- Run a sample query on DynamoDB using the catalog name:
![88](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/c4c3dd41-6149-4f48-a6c7-3813ec5542c7)
- Run the query below which joins (Inner join) the trip data from Aurora PostgreSQL with DynamoDB. We have used the `riderid` attribute from DynamoDB to join with `rider_email` column of
`trips_query` table in Aurora PostgreSQL. `trips_info` field is used as an additional join condition. The purpose of the query is to check data consistency of trip record between the two data stores.
``` SQL
SELECT ddb.riderid,ddb.tripinfo , ddb.fare_amount "DDB-Fareamount", rdb.fare_amount "RDB-Fareamount", ddb.tolls_amount "DDB-Tollsamount", rdb.tolls_amount "RDB-Tollsamount", ddb.passenger_count "DDB-passenger_count", rdb.passenger_count "RDB-passenger_count", ddb.tip_amount "DDB-Tipamount", rdb.tip_amount "RDB-Tipamount",  ddb.total_amount "DDB-Totalamount", rdb.total_amount "RDB-Totalamount"
FROM
ddbcatalog.default."aws-db-workshop-trips" ddb,
rdbcatalog.public.trips rdb
where 
ddb.riderid=rdb.rider_email
and ddb.tripinfo=rdb.trip_info;
```
![jjjj](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/f39f3b24-cf2e-42e6-afe1-26947f61130c)
The trip record between Amazon DynamoDB and Amazon Aurora is consistent!



























