# Overview 

In this lab, we are going to set up a VPC with public and private subnets, connect it to the internet and establish private connections to AWS services.

![vpc-project](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/57617ff1-8447-47f7-9f6f-b431e4f51e15)

# Prerequisites
Before starting the workshop, the following ressources should be created:
- IAM role for EC2 instances. The `AmazonS3FullAccess` is required to complete the VPC Endpoints section and `AmazonSSMManagedInstanceCore` allows us to remotely connect to the instance using Session Manager instead of SSH
- EC2 instance profile to define the IAM role that EC2 instances should use.
- IAM role for VPC Flowlogs
- An S3 bucket with the name `networking-day-${AWS::Region}-${AWS::AccountId}` that will be used to test VPC Endpoints

This is achieved by using the following CloudFormation template:
<details open>
<summary>pre-requisites.yaml</summary>
  
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
</details>

Create a new stack on CloudFormation named `NetworkingWorkshopPrerequisites` and uplaod to it the `pre-requisites.yaml` template file. Once the stack status becomes `CREATE_COMPLETE`, you're all set to start the lab.

![cloudformation_complete](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/c50a427d-6dc6-4f47-a480-035e1bbb1861)

# Amazon VPC
Let's begin by creating a VPC with the following settings:
- Name tag: `VPC A`
- IPv4 CIDR block: `10.0.0.0/16`
- Tenancy: `Default`
- IPv6 is disabled.

After that, edit the VPC settings to enable DNS hostnames.

![vpc-settings](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/ef8207b6-d0ed-4449-8e42-435455aa304b)

# Subnets
Let's proceed by creating two public and two private subnets in each of two availability zones within VPC A with the following settings:

**Subnet 1**
 - Name: `VPC A Public Subnet AZ1`
 - Availability zone: `us-east-1a`
 - IPv4 subnet CIDR block: `10.0.0.0/24`
   
**Subnet 2**
 - Name: `VPC A Private Subnet AZ1`
 - Availability zone: `us-east-1a`
 - IPv4 subnet CIDR block: `10.0.1.0/24`
   
**Subnet 3**
 - Name: `VPC A Public Subnet AZ2`
 - Availability zone: `us-east-1b`
 - IPv4 subnet CIDR block: `10.0.2.0/24`
   
**Subnet 4**
 - Name: `VPC A Private Subnet AZ2`
 - Availability zone: `us-east-1b`
 - IPv4 subnet CIDR block: `10.0.3.0/24`

![subnets](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/90d8e656-9d13-4b6e-aa0f-27e86d62cb63)

# Network ACLs
We can add a layer of security for VPC A by using a network access control list (ACL) for controlling traffic in and out of one or more subnets.
The VPC has already a default NACL, but we will still create a new one alongside it just to demonstrate the process. It will be named `VPC A Workload Subnets NACL`and be associated to all 4 subnets of VPC A.

![subnets-nacl](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/b1f75643-4779-4e6c-84fd-b8c0b99d6635)

Because NACLs are created with only a DENY rule for inbound and outbound we will now change the default NACL rules to allow all traffic in both directions. We will do so by adding a new rule numbered `100` with `All traffic`as type and `0.0.0.0/0` as source if it's an inbound rule or as a destination if it's an outbound rule.

# Route Tables
We need to create a new public route table for the public subnets with a route to the internet via the Internet Gateway. It has the name of `VPC A Public Route Table`.

![public-rt](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/c6bc3e68-8572-423e-b206-e6cec9179118)

Similarly, we need to create a new private route table for the private subnets with a route to the internet via NAT Gateway. It has the name of `VPC A Private Route Table`.

![privatert](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/0df9f805-804d-4c18-9ea4-94f77f50a021)

# Internet Connectivity
Let's create an Internet Gateway named `VPC A IGW`and attach it to VPC A.

![igw](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/5a9a5f75-fa31-4f72-bdda-49892d7df594)

In order to utilize the newly created Internet Gateway, we need to update VPC routing tables to point the default routes for our public subnets to this Internet Gateway. The new route has ` 0.0.0.0/0` as destination and `VPC A IGW` as target.

![igwroute](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/205a9186-971e-4792-9c87-0c531cc4e636)

Next we will add outbound connectivity from the private subnets by deploying a NAT Gateway in a public subnet for use by workloads that should not be directly exposed to the internet. It is named `VPC A NATGW` and resides in the `VPC A Public Subnet AZ1` subnet. An elastic IP address is allocated to it.
We now add a route entry to the Route Table for the private subnets. It has a destination of `0.0.0.0/0` and `VPC A NATGW` as target.

![igwprivate](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/b6901768-4a75-4217-a497-715fb4d66f6b)

# VPC Endpoints
We need to create an Interface endpoint to use it with the AWS KMS service. Let's name it `VPC A KMS Endpoint` with KMS chosen as a service and `VPC A` as a VPC with the private subnets `VPC A Private Subnet AZ1`and `VPC A Private Subnet AZ2`. Make sure the defaut security group is selected and the policy left as `Full Access`.

Let's create another Endpoint for the S3 bucket. This time, the name is `VPC A S3 Endpoint` and the service is S3 with the gateway type. VPC A is selected as well as both route tables. The policy is again left to `Full Access`.

![endpoints](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/894ef65e-be2d-4cef-aba4-cc5fb22689dc)

# EC2 Instances
We will first create an EC2 instance in the Public Subnet of AZ2 with a name of `VPC A Public AZ2 Server` and instance type of `Amazon Linux 2023 t2.micro`. We won't need a key pair since we will be using Systems Manager to connect to the instances. Make sure the VPC selected is VPC A, the subnet is `VPC A Public subnet AZ2` and a public address is set to be auto-assigned.
Afterwards, a security group of name `VPC A Security Group` and a description of `Open-up ports for ICMP` is created. A new rule is added with the type `All ICMP - IPv4` and source `0.0.0.0/0`. The goal is to only allow ICMP traffic to reach the hosts.
Under the `Advanced network configuration` section, add `10.0.2.100` as a primary address. And under the `Advanced details`, select `NetworkWorkshopInstanceProfile` we created in the pre-requisites step.

![publicec2](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/e11e7b87-9342-45d7-bdd7-43de97de0623)

To create another EC2 instance in the private subnet with the same sittings of the public EC2 instance, we just select the latter then choose `Launch more like this` from the `Image and templates` menu. Just change the name to `VPC A Private AZ1 Server` and the subnet to `VPC A Private Subnet AZ1`. In this case, the `Auto-Assign Public IP` is disabled. Configure the primary IP address to be `10.0.1.100`.

# Test Connectivity
In this step, we're going to test the connectivity of our network. 
First copy the public IP address of the public EC2 instance and ping it from a terminal. 

![ping public](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/8b7fa095-0687-4c1c-a0b2-046186622588)

Now connect to the private EC2 instance using the Session Manager. Then ping the private IP address of the pulic EC2 instance and `example.com`.

![pingprivate](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/9feaf389-2319-404e-a9b2-7e9a7277fae4)

Let's check the DNS for the KMS service from the VPC A instance:

![diganswer](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/756f7878-21be-47dc-ab1b-0b741de95b4e)

As expected, The response points to two local IP addresses within the VPC A private subnet CIDR blocks of 10.0.1.0/24 and 10.0.3.0/24. This means that the traffic to KMS from VPC A will be routed to the VPC Endpoint rather than traversing the internet to reach KMS.

# Clean Up
Before finishing the lab, don't forget to do the clean up i.e. terminate the EC2 instances, delete the VPC endpoints, delete VPC A and the CloudFormation stack.

 


