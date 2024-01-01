# Overview
In this workshop, we'll explore the basics of event-driven design and the main AWS services that enable building event-driven, loosely-coupled and distributed architectures like: Amazon 
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
An event rule can also route to multiple targets at once. In this step, we will set 3 targets: Amazon API Gateway endpoint, AWS Step Function state machine and Amazon Simple Notification Service 
(Amazon SNS) topic, which are already provisionned by the CloudFormation stack.
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





































