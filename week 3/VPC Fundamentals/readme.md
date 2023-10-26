# Prerequisites
Before starting the workshop, the following ressources should be created:
- IAM role for EC2 instances. The `AmazonS3FullAccess` is required to complete the VPC Endpoints section and `AmazonSSMManagedInstanceCore` allows us to remotely connect to the instance using Session Manager instead of SSH
- EC2 instance profile to define the IAM role that EC2 instances should use.
- IAM role for VPC Flowlogs
- An S3 bucket with the name `networking-day-${AWS::Region}-${AWS::AccountId}` that will be used to test VPC Endpoints

This is achieved by using the following CloudFormation template:
```
AWSTemplateFormatVersion: "2010-09-09"
Description: "Create IAM roles for EC2 instances and Flow Logs and an S3 Bucket for endpoint policy tests"

Resources:
  # Account Level Resources
  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      RoleName: "NetworkingWorkshopEC2Role"
      Path: "/"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/AmazonS3FullAccess
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ec2.amazonaws.com
            Action:
              - sts:AssumeRole

  EC2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      InstanceProfileName: "NetworkingWorkshopInstanceProfile"
      Path: "/"
      Roles:
        - !Ref EC2Role

  FlowLogsRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: NetworkingWorkshopFlowLogsRole
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - vpc-flow-logs.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Description: Role to allow VPC Flow Logs to write to CloudWatch logs
      Policies:
        - PolicyName: CloudWatchLogsWrite
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: 
                  - "logs:CreateLogGroup"
                  - "logs:CreateLogStream"
                  - "logs:PutLogEvents"
                  - "logs:DescribeLogGroups"
                  - "logs:DescribeLogStreams"
                Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:NetworkingWorkshopFlowLogsGroup:*'

  GatewayEndpointBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'networking-day-${AWS::Region}-${AWS::AccountId}'
      LifecycleConfiguration:
        Rules:
          - ExpirationInDays: 3
            Status: Enabled
```
