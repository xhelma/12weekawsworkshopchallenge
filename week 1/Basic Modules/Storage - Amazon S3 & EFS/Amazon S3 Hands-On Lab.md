# Amazon S3 Hands-On Lab
This lab is designed to demonstrate how to interact with S3 through the AWS Console and how to setup access to objects within a private S3 bucket to be viewed through an EC2 web host.

## Prerequisites
A web host, used to interact with the S3 bucket, is built through the following CloudFormation template:

<details>
  <summary>S3-General-ID-Lab.yaml </summary>
  
```
AWSTemplateFormatVersion: 2010-09-09
Description: This CloudFormation template will produce an EC2 Web Host
Parameters:
  AmiID:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Description: The AMI ID - Leave as Default
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'
  InstanceType:
    Description: Web Host EC2 instance type
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - m5.large
  MyVPC:
    Description: Select Your VPC (Most Likely the Default VPC)
    Type: 'AWS::EC2::VPC::Id'
  MyIP:
    Description: Please enter your local IP address followed by a /32 to restrict HTTP(80) access. To find your IP use an internet search phrase "What is my IP".
    Type: String
    AllowedPattern: '^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])(\/(32))$'
    ConstraintDescription: Must be a valid IP CIDR range of the form x.x.x.x/32
  PublicSubnet:
    Description: Select a Public Subnet from your VPC that has access to the internet
    Type: 'AWS::EC2::Subnet::Id'

Resources:
  WebhostSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      VpcId: !Ref MyVPC
      GroupName: !Sub ${AWS::StackName} - Website Security Group
      GroupDescription: Allow Access to the Webhost on Port 80
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '80'
          ToPort: '80'
          CidrIp: !Ref MyIP
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName} - Web Host Security Group
  WebServerInstance:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: !Ref AmiID
      InstanceType: !Ref InstanceType
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}
      NetworkInterfaces:
        - GroupSet:
            - !Ref WebhostSecurityGroup
          AssociatePublicIpAddress: 'true'
          DeviceIndex: '0'
          DeleteOnTermination: 'true'
          SubnetId: !Ref PublicSubnet
      UserData: 
        Fn::Base64:
          !Sub | 
          #!/bin/sh
          yum -y update
          yum -y install httpd
          amazon-linux-extras install php7.2
          yum -y install php-mbstring
          yum -y install telnet
          case $(ps -p 1 -o comm | tail -1) in
          systemd) systemctl enable --now httpd ;;
          init) chkconfig httpd on; service httpd start ;;
          *) echo "Error starting httpd (OS not using init or systemd)." 2>&1
          esac
          if [ ! -f /var/www/html/s3-web-host.tar.gz ]; then
          cd /var/www/html
          wget https://workshop-objects.s3.amazonaws.com/general-id/s3_general_lab/s3-web-host.tar
          tar xvf s3-web-host.tar
          chown apache:root /var/www/html/labs/s3/s3.conf.php
          chown apache:root /var/www/html/labs/s3/reset_config/s3.conf.php
          fi
          yum -y update
Outputs:
  PublicIP:
    Value: !Join 
      - ''
      - - 'http://'
        - !GetAtt 
          - WebServerInstance
          - PublicIp
    Description: Newly created webhost Public IP
  PublicDNS:
    Value: !Join 
      - ''
      - - 'http://'
        - !GetAtt 
          - WebServerInstance
          - PublicDnsName
    Description: Newly created webhost Public DNS URL
```
</details>

Browsing to the public IPv4 address of the web host displays the "S3 Hands-On Lab" page.

![s3-site](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/01b60ecf-28f7-4e8f-8453-5fd48624dd52)

## Creating a bucket in S3
We will start off by creating a S3 bucket in the same region where we set up the web host with CloudFormation. The bucket will be private so `Block all public access` is kept checked.

![xhem-s3](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/acf26b4e-e1e3-4cac-9a09-1ef04ef7291a)

## Adding objects to your S3 bucket
Next, we're going to uplaod 7 photos into the bucket.

![pics7in](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/b33f173e-c43c-450b-b5d8-90bcfe3582bd)

## Working with objects in the S3 Console
There are many file system commands that we can perform on S3 objects using the GUI such as renaming, moving, copying, deleting; to name a few.  
In this step, we consider the case scenario of moving `photo7.jpg` to a new folder (prefix).  
First, create a folder named `photo7`. Then move the `photo7.jpg` object to it.

![ph7mvd](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/321ed812-b5fe-43e0-8fff-c9131a3244a3)

## Accessing objects stored in S3
In order to view S3 objects hosted in the bucket from the web host, we need to give it permissions to access the bucket. Since our S3 bucket is private, we will do this by creating a policy 
and attaching it to a role with the Identity and Access Management (IAM) service.  
Select the S3 service, allow the `GetObject`action only and the S3 bucket ARN.

![s3policyy](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/b76e3817-f864-4f4a-8cf4-a71fb96e5bd9)

Create a new role with the `Trusted entity type` being the EC2 AWS service. Then add the policy previously created as the permission policy. Lastly, attach the role to the web host.
![ec2tos2iam](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/49122d16-06c1-4c6f-be4c-c6512c4cdf9a)

From the web host landing host, enter the name of the bucket and the region it resides in to view the objects.
![pics-on-site](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/89380b24-870d-4701-9306-b530ec5945ac)

## Enabling bucket versioning
Enable bucket versioning on the bucket as a first step. Then, we're going to upload a new `photo1.jpg` object. Refresh the "S3 Hands-On Lab" page, we now see the 
new version of `photo1.jpg` no longer covered by a red X. Note that the older version of the object remains in the bucket if it needs to be downloaded or restored later.
![no x](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/4f3743d6-1dff-4817-8ab0-4a31359b8fea)

## Setting up a Lifecycle Policy
We are going to setup a lifecycle policy that will move non-current versions of the objects to the S3 Infrequent Access (IA) tier after 30 days and then delete them 30 days later.  
Create a lifecycle rule that will be applied bucket-wide and check `Move noncurrent versions of objects between storage classes` and `Permanently delete noncurrent versions of objects`.
Choose `Standard-IA` as the storage class that non-current versions will be transitioned into and `30` as the number of days after which the transition takes place. Set the period after
which non-current objects will be permanently deleted to 60 days.
![lifecycle](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/69061ce5-ba22-4343-ae67-7b938de853ef)


## Cleanup
Empty the bucket first then delete it. Delete the CloudFormation stack afterwards.



















