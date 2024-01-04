# Overview
In this workshop[^1], we'll explore the basics of event-driven design and the main AWS services that enable building event-driven, loosely-coupled and distributed architectures like: Amazon 
EventBridge, Amazon SNS and Amazon SQS.

# Prerequisites
Before starting out, we'll deploy the following CloudFormation in the `us-east-1` region:
<details>
<summary>master-v2.yaml</summary>  
  
```
---
AWSTemplateFormatVersion: "2010-09-09"
Description: "Master stack: AWS Event-driven Architectures Workshop"

Resources:
  CognitoUserPool:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://aws-event-driven-architecture-workshop-assets.s3.amazonaws.com/cognito-user-pool-v2.yaml
      TimeoutInMinutes: 60

  SNS:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://aws-event-driven-architecture-workshop-assets.s3.amazonaws.com/sns-v2.yaml
      TimeoutInMinutes: 60

  EventBridge:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://aws-event-driven-architecture-workshop-assets.s3.amazonaws.com/event-bridge-v2.yaml
      TimeoutInMinutes: 60

  Lambda:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://aws-event-driven-architecture-workshop-assets.s3.amazonaws.com/lambda-v2.yaml
      TimeoutInMinutes: 60

Outputs:
  StackRef:

    Value: !Ref CognitoUserPool

  EventGeneratorConfigurationUrl:
    Description: Event Generator configuration link
    Value: !GetAtt CognitoUserPool.Outputs.EventGeneratorConfigurationUrl

  WildRydesSaasPlaygroundConfigurationUrl:
    Description: Wild Rydes Saas Playground configuration link
    Value: !GetAtt CognitoUserPool.Outputs.WildRydesSaasPlaygroundConfigurationUrl

  CognitoUsername:
    Description: Cognito username for use with Event Generator and Wild Rydes SaaS Playground
    Value: !GetAtt CognitoUserPool.Outputs.CognitoUsername

  CognitoPassword:
    Description: Event Generator password for use with Event Generator and Wild Rydes SaaS Playground
    Value: !GetAtt CognitoUserPool.Outputs.CognitoPassword

  ApiUrl:
    Description: "API Gateway endpoint URL"
    Value: !GetAtt EventBridge.Outputs.ApiUrl
```
</details>

# Event-driven with EventBridge
## First event bus and targets
Let's create our first event bus called `Orders`, as well as an EventBridge rule, `OrderDevRule`, which matches all events sent to the `Orders` event bus and sends the events to a CloudWatch Logs 
log group, `/aws/events/orders`.
![eb_arch_simple_bus](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/069097f7-1373-4fa0-bfec-bb5c3dc4d58d)
Navigate to the Amazon EventBridge service console and create a new event bus called `Orders` with default settings. After selecting this event bus, create a rule named `OrdersDevRule`
with a custom event source using the following JSON event pattern:
```json
{
   "source": ["com.aws.orders"]
}
```
Set the rule target as a CloudWatch log group and name it `/aws/events/orders`.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/28343c7b-6f4e-47d3-8c02-324683fbe08f)
Now we'll send an event to test the rule. Set the event source to `com.aws.orders` with the detail type `Order Notification` and the following JSON payload for the event detail:
```json
{
   "category": "lab-supplies",
   "value": 415,
   "location": "eu-west"
}
```
From the Cloudwatch log group console, we can verify that we have indeed received the event.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/417e7a6a-a719-4ddc-b5d7-86f7fe190db2)

## Working with EventBridge rules
An event rule can also route to multiple targets at once. In this step, we will set 3 targets: Amazon API Gateway endpoint, AWS Step Function state machine and Amazon Simple Notification Service (Amazon SNS) topic, which are already provisionned by the CloudFormation stack.
![eb_rules_2](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/217069d9-db9a-4e53-9d76-47ecaab005ae)

Let's create an EventBridge API destination with the name `api-destination` and add in the API URL found in the output of the CloudFormation stack as a value of the key `ApiUrl`. Select 
`POST` as the HTTP method and create a new connection called `basic-auth-connection` with `Basic (Username/Password)` as the authorization type. The username is `myUsername` and the
password is `myPassword`.
We'll configure an EventBridge rule to target the EventBridge API Destination. We name it `OrdersEventsRule` with the same event source as before but with the `api-destination` as target.
To test the rule, we'll send the following `Order Notification` event from the source `com.aws.orders`:
```json
{ "category": "lab-supplies", "value": 415, "location": "us-east" }
```
From the `API-Gateway-Execution-Logs` CloudWatch log group, we can verify the basic authorization was successful and the API responds with a 200 status.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/579a9b84-664a-4a64-917a-a241155a1f8b)
Next, we'll create another rule named `EUOrdersRule` that process only orders from locations in the EU (eu-west or eu-east) with an order value greater than 1000 and targets the 
`OrderProcessing` Step Function state machine. Use the following event pattern:
```json
{
  "source": ["com.aws.orders"],
  "detail": {
    "location": [{
      "prefix": "eu-"
    }],
    "value": [{
      "numeric": [">", 1000]
    }]
  }
}
```
Send the following two `Order Notification` events from the source `com.aws.orders` to test our rule:
```json
{ "category": "office-supplies", "value": 300, "location": "eu-west" }
```
```json
{ "category": "tech-supplies", "value": 3000, "location": "eu-east" }
```
We can verify than only the second event got matched and triggered the execution of the step function state machine:
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/9ab906a8-6d45-4f10-829c-92103553617c)

Lastly, we'll create a rule named `USLabSupplyRule` that only process events of orders from US locations (us-west or us-east) that are `lab-supplies` and targets the `Orders` SNS topic.
Use the following event pattern:
```json
{
  "source": ["com.aws.orders"],
  "detail": {
    "location" : [{ "prefix": "us-"}],
    "category" : ["lab-supplies"]
  }
}
```
Send the following two `Order Notification` events from the source `com.aws.orders` to test our rule:
```json
{ "category": "lab-supplies", "value": 415, "location": "us-east" }
```
```json
{ "category": "office-supplies", "value": 1050, "location": "us-west", "signature": [ "John Doe" ] }
```
Only the first event matches, which is then sent to the Orders SQS Queue via Orders SNS Topic. Poll for messages to verify the received one.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/25216e81-9b35-435b-a41a-e435406ce4c9)

## Scheduling expressions for rules
We can create rules that self-trigger on an automated schedule in EventBridge Scheduler using cron or rate expressions. Here, we'll demonstrate cerating a schedule using a cron expression
since it offers more granular options.
Create a new schedule with the name `OrdersProcessing` that is recurrent and cron-based with the expression `* * ? * MON-FRI *`; which means a message to be delivered every minute, 
Monday through Friday. The target will be the `Orders` event bus with `com.aws.orders` as source and `scheduled-event` as detail type. The following is the message body we will be 
sending to the event bus:
```json
{ "category": "lab-supplies", "value": 500, "location": "us-east" }
```
Checking the CloudWatch `/aws/events/orders` log group, we can see that a log stream is created every minute.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/03aa57c8-1f07-4be9-9b8b-323034ede77d)
Edit the schedule ending time to put a stop to the incoming events.

## Working with SaaS partners
In this section we will walk through how to set up the `wildrydes.com` partner event source, create a partner event bus, and send test events from the EventBridge SaaS Integration 
Playground.
![playground_arch](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/f5805dfd-8cca-4fa4-a131-2bdfc3e57a3e)
From the CloudFormation stack outputs, Open the `WildRydesSaasPlaygroundConfigurationUrl` hyperlink. Complete the configuration of the Cognito User pool by adding the username and password.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/9a098ad1-cd00-43b8-b322-aebcf7b344ba)

Create an event source in the `us-east-1` region. From the EventBridge console, navigate to the Partner event sources and find the `wildrydes.com` event source. Choose Associate with an 
event bus. Then create a rule that targets the `aws/events/playground` CloudWatch log group.
Let's test partner events by choosing to send a stream of Shopify events. We can see the corresponding logs from the CloudWatch log group:
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/51fc8c87-3746-4f12-8545-3963741ec489)
## Using the schema registry
Now that we have Order events getting published to our EventBridge custom event bus, we will enable Schema Discovery to automatically get schema for those events and we will generate code bindings that will be used to implement a Lambda business logic.
![eb_schema_arch](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/2a4644b4-0570-473d-b10f-322296df5694)

Let's start first by enabling schema discovery on the `Orders` event bus. Then, we'll publish a test event from the Cloud9 environment deployed by the CloudFormation stack.
In the root directory, create a folder called `Events` that contains a file with the name `OrderNotification_v1.json` and has the following JSON test event:
```json
[
  {
    "EventBusName": "Orders",
    "Source": "com.aws.orders",
    "Detail": "{ \"category\": \"lab-supplies\", \"value\": 415, \"location\": \"eu-west\" }",
    "DetailType": "Order Notification"
  }
]
```
From the terminal, run this script to publish the test event:
```sh
cd Events
aws events put-events --entries file://OrderNotification_v1.json
```
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/5929b732-193b-454f-8556-21b4867d668d)
From the discovered schema registry tab, we can see a schema for Order Notification event that we published earlier. Next we'll generate a sample Lambda function using Amazon SAM CLI.
Enter the SAM service shell using the command `sam init` then choose the `Infrastructure event management` as the Quick Start application template and select the `python 3.11` runtime.
Since we are going to consume our previously discovered Order Notification event, the starter template will be an EventBridge App from scratch (100+ Event Schemas). X-ray tracing and
CloudWatch Application Insights monitoring are not necessary for this app. Name the project `forecast-service` and keep the default AWS profile and our event AWS region. We want to use the newly discovered schema in our project.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/ec76644e-0475-47a5-a8ed-d16cfe8ce9de)
Edit the AWS SAM `template.yaml` file found in the `forecast-service` folder to change the event bus name from default to `Orders` and add the following statement to the `template.yaml` file after our Lambda function definition to define a DynamoDB table to store event details for our forecast service.
```yaml
# DynamoDB Orders table definition
OrderDetailsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: OrderDetails
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: orderId
        AttributeType: S
    KeySchema:
      - AttributeName: orderId
        KeyType: HASH
```
Our business logic will store event details in DynamoDB table, as well as publish a new event when Order Notification event is successfully processed and data is available for Third-Party service. So the Lambda function needs extra permissions:
```yaml
      Policies:
      - DynamoDBWritePolicy:
            TableName: !Ref OrderDetailsTable
      - EventBridgePutEventsPolicy:
            EventBusName: Orders
```
Open the `app.py` file in the `forecast-service/hello_world_function/hello_world` folder and insert this code block before Lambda handler definition - `def lambda_handler(event, context):` line, to add the required Python libraries imports and initialize DynamoDB and EventBridge clients from AWS SDK:
```py
# AWS SDK(boto3) and other libraries imports
import boto3
import json
import uuid
import datetime

# DynamoDB client
dynamodb = boto3.resource('dynamodb')
orderDetailsTable = dynamodb.Table('OrderDetails')

# EventBridge client
eventBridgeClient = boto3.client('events')
```
We also need to add a business logic to our Lambda handler. Replace `#Execute business logic` line with the following code block:
```py

    # Print Order Notification event details
    print(detail)

    # New order Id
    orderId = str(uuid.uuid4())

    # Save Order Notification details into DynamoDB table
    response = orderDetailsTable.put_item(
       Item={
             'orderId': orderId,
             'category': detail.category,
             'location': detail.location,
             'value': str(detail.value)
       }
    )
    print(response)

    # Publish Order Processed event
    response = eventBridgeClient.put_events(
       Entries=[
             {
                'Time': datetime.datetime.utcnow(),
                'Source': 'com.aws.forecast',
                'DetailType': 'Order Processed',
                'Detail': json.dumps({'orderId': orderId}),
                'EventBusName': 'Orders'
             }
       ]
    )
    print(response)
```
It's time to deploy the Lambda function project using the `sam deploy -g` in the terminal with the following settings:
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/4ca4c658-babf-4e20-8a7f-d0ecb8d2c157)
To test our business logic, we need to publish Order Notification event. To do it, run the following script:
```sh
cd ../
cd Events
aws events put-events --entries file://OrderNotification_v1.json
```
From the CloudWatch log group, we can see that the Lambda function was indeed invoked.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/49e7e644-a5f5-47ab-9d1f-fa014a085f51)
We can also verify that the our Order Notification event details got stored in the DynamoDB table `OrderDetails`:
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/3d2a3b4a-fe93-4242-9d51-2ddc4e8b0cdd)
## Archive and Replay
In this section, we will enable archive for `Orders` custom event bus to match an event with a `com.aws.orders` source, and then replay historical order events to a new SQS queue.
![eb_replay_arch](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/840b5ef3-2d31-4f3a-ba80-15cda0ea0414)

Create an archive with the name `OrderEventArchive` with a retention period of 30 days. Set filtering on events that match the following pattern:
```json
{
    "source": [
        "com.aws.orders"
    ]
}
```
Let's test archiving by sending the following event payload to the `Orders` event bus:
```json
    {
      "category": "office-supplies",
      "value": 1200,
      "location": "eu-west"
    }
```
We can check the archive again to verify that the event count and size have been updated.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/48a34d41-c244-4e84-92a1-28a49193624d)
Now, we'll create an EventBridge rule called `OrdersReplayRule` for the replayed archive that targets the SQS queue `OrdersReplayQueue` using the following event pattern:
```json
{
  "replay-name": [{
    "exists": true
  }],
  "source": ["com.aws.orders"]
}
```
Start a new replay under the name of `OrdersReplay` that is sourced from the archive and destined to the `Orders` event bus and applying the previously created rule `OrdersReplayRule`.
![aws](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/eed0907d-f839-420d-b8d7-9aad5c393f00)
Navigate to the `OrdersReplayQueue` SQS queue and poll for received messages. We can see messages in the queue which are replayed archive order events.
![Screenshot 2024-01-02 at 11-52-39 Send and receive messages Simple Queue Service us-east-1](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/c8be25d2-f58d-4875-9392-bcdd7ccb8015)
## Global endpoints
In this section, we'll make our event-driven application more available with the help of a global endpoint which allows failover to a secondary region.

![eb_globalendpoint_setup_arch](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/7999ae7b-9a16-4dc5-92d1-eeedd8790735)

Let's start by creating a second event bus with the same name as the first one but in the `us-west-2` region. A corresponding rule for CloudWatch is also created.
Then, we'll create a global endpoint in the primary region with the name `OrdersGlobalEndpoint` that uses a Route 53 health check and a CloudWatch alarm that are deployed using a CloudFormation stack. We'll keep the defaults in the template. Make sure that event replication is enabled.
![capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/a32a2384-64a6-4ac4-acc8-19cf2a7b6f7c)

Next, we'll use the `PutEvents` API to publish test events to the global endpoint from AWS CLI. To do so, open the Cloud9 environment and the following script after replacing
with our endpoint ID:
```sh
cd Events
aws events put-events --entries file://OrderNotification_v1.json --endpoint-id xxxxxxxxxx.xxx
```
Checking the CloudWatch `/aws/events/orders` log group in both regions, we can see that the published events have identical resource fields, which points to the global endpoint.
Lastly, we'll try testing the global endpoint failover feature by inverting the Route 53 health check status to unhealthy. We'll repeat publishing the events through the PutEvents API and checking the CloudWatch logs again shows us that events got routed to the secondary region `us-west-2`.
![capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/96f3d552-ffc2-44d8-aac8-d8a9e1522cee)

# Event-driven with Lambda
We'll configure the `Inventory` event bus as a successful Lambda Destination on `InventoryFunction` Lambda function, when it is asynchronously invoked and successfuly executed.
![capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/0b892b72-2812-4a55-9ecd-ef34348cc0dc)

Next, we'll create a rule on the `Orders` event bus called `OrderProcessingRule` that routes the event published by the step function to the `Inventory` function. The rule pattern used is the following:
```json
{
    "source": [
        "com.aws.orders"
    ],
    "detail-type": [
        "Order Processed"
    ]
}
```
To test the end-to-end functionality, we're going to publish a message to the Orders EventBridge event bus with the following payload:
```json
{
  "category": "office-supplies",
  "value": 1200,
  "location": "eu-west",
  "OrderDetails": "completed"      
}
```
From the CloudWatch `/aws/events/inventory` log group, we can find the log stream of this event:
![capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/9170161b-d39e-45bf-ad38-6aebca89586b)

We will now handle the case of a failure in the execution of the `InventoryFunction` Lambda function by creating a Lambda Destination to a SQS queue of the name `InventoryFunctionDLQ`.
![capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/1bc792d4-e6cd-40d7-b63b-4f581acd11d0)
Similar to the success scenario case, we will test the end-to-end functionality of the failure one by publishing a message to the Orders EventBridge event bus with the following payload:
```json
{
  "category": "office-supplies",
  "value": 300,
  "location": "eu-west",
  "OrderDetails": {
    "force-error": true
  }
}
```
On the SQS page, we can see that the `InventoryFunctionDLQ` has one message available:
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/cce91066-b748-47e5-81c9-c49b5b46e249)

# Event-driven with SNS
In this section, we will build a simple pub/sub implementation using Amazon SNS as our publishing service and Amazon SQS as a subscriber. This will allow us to easily verify the successful delivery of the messages.

We'll start by creating the SQS queue called `OrdersQueue` that has a standard type.
To receive messages published to a topic, we must subscribe an endpoint to it. In this case, we will subscribe the SQS queue that we created in the previous step as the endpoint to SNS topic.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/29b351d4-6b33-4167-8445-f848f89abb1c)

To verify the result of the subscription, you will use the Publish Message functionaliy to publish to the topic and then view the test message that the topic sends to the queue. As a result, the `OrdersQueue` has one available message visible.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/558a745d-7c2b-4fab-b158-8857f20265c8)

We now want to implement message filtering on the SNS topic so that each subsciber only receives those relevant to it. We'll demonstrate that by creating a new standard SQS queue for EU Orders, that stores the messages filtered out with the `location` attribute set to `eu-west`. We call it `Orders-EU` and subscribe it to the `Orders` SNS topic.
![capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/8ae6075b-fed2-4a12-badd-e32137b248a9)
From the SNS topic console, edit the `Orders-EU` SQS queue to enable a filtering policy based on the follwing message attribute:
```json
{
"location": ["eu-west"]
}
```
Similarly; we'll create another SQS queue for book orders, with the name of `BookOrders`. This time filtering happens on the level of the message body that contains the attribute of `category`. Edit it from the SNS topic console to add the following body message filter:
```json
{
"category": ["books"]
}
```
Now, we'll test the sysrem behavior by publishing different kind of messages to the Orders SNS topic:
- A message with a plain text body and no attribute.
- A message with a plain text body and a `location` attribute with the value `us-west` which doesn't match the filter.
- A message with a plain text body and a `location` attribute with the value `eu-west` which does match the filter.
- A message with no attributes and the following non-matching message body:
```json
{
   "category": "groceries",
   "value": 555
}
```
- A message with no attributes but with a matching "books" category in its body:
```json
{
"category": "books",
"value": 70
}
```
Checking the SQS queues, we can see that all 5 messages are delivered to the `Orders` queue, while the `Orders-EU` and the `BookOrders` queues shows 1 message each. 
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/48099d0a-60a4-4caa-a757-6d6468792e5f)

Let's experiment further with advanced message filtering by creating an SQS queue called `Orders-XL`, to store extra-large orders with a quantity of 100 and greater and subscribing it to the `Orders` SNS topic. The subscription filter policy is as follows:
```json
{
  "quantity": [
    {
      "numeric": [">=", 100]
    }
  ]
}
```
We'll create a similar SQS queue that stores the same type of messages with the added condition that they originate from the EU. We name it `Orders-XL-EU`, subscribe to 
the `Orders` SNS topic and add to it the following filter policy:
```json
{
  "location": [{ "prefix": "eu" }],
  "quantity": [
    {
      "numeric": [">=", 100]
    }
  ]
}
```
Let's test that the filtering is working as it should by sendig 3 messages:
- A message with a plain text body and a `quantity` attribute with the value `50` which doesn't match the filter.
- A message with a plain text body and a `quantity` attribute with the value `100` which does match the filter.
- A message with a plain text body and a `location` attribute with the value `eu-west` and a a `quantity` attribute with the value `100`.

As a result, we see the `Orders` queue having 3 messages delivered, the `Orders-EU` and `Orders-XL-EU` queues showing 1 message delivered and `Orders-XL` with 2 messages delivered.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/9084ab98-4609-454b-8c49-8b8dba246a7d)


Finally, we'll handle the case of message delivery fails by creating a dead-letter SQS queue, called `OrdersTopicDLQ`, to act as the Dead Letter Queue for messages that the `Orders` SNS topic is unable to deliver to subscriptions configured with the DLQ.
We need to grant permissions to the `Orders` SNS topic so that it can send messages to the `OrdersTopicDLQ` SQS queue. For that, append the following policy statement to the one present on the access policy tab of the `OrdersTopicDLQ` SQS queue after substituting with our ressources' ARN:
```json
{
  "Sid": "topic-subscription-sns",
  "Effect": "Allow",
  "Principal": {
    "AWS": "*"
  },
  "Action": "SQS:SendMessage",
  "Resource": "OrdersTopicDLQ_ARN",
  "Condition": {
    "ArnLike": {
      "aws:SourceArn": "SNS_ARN"
    }
  }
 }
```
We'll navigate to the `DodgyFunction` Lambda function and add configure it so that it gets triggered by the `Orders` SNS topic. 
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/9f4892d1-2156-48af-bf36-48e33e35dbc7)
Let's verify the Lambda subscription by sending out a test message to the `Orders` SNS topic. From the monitoring pane on Lambda function detail page, we can confirm that the
function was invoked successfully one time.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/d1802a7b-15ca-4945-9b0b-3ce9686096d6)
From the `Orders` SNS topic, edit the `OrdersTopicDLQ` subscription so that the redriving policy to the DLQ is enabled while specifying the `OrdersTopicDLQ` as destination.


To make message deliveries fail, we'll delete the `DodgyFunction` Lambda function. Then, resend another test message as in the previous step.
We observe that the `Orders` queue shows both messages delivered, while the `OrdersTopicDLQ` only received one in case of delivery failure because of the Lambda function deletion.
![Capture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/1e83c215-7c85-4c2f-aefc-a0ec4e9b2470)


[^1]: [Building event-driven architectures on AWS Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/63320e83-6abc-493d-83d8-f822584fb3cb/en-US)


























































