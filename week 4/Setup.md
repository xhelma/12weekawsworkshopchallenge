As a setup, we're going to need to deploy the necessary ressources via AWS CloudFormation.
Let's start by creating a bucket in the `us-west-2` region that will be hosting the CloudFormation templates. Upload to it the `Base Template` and `S3 Best Security Practices` template
under the `assets` prefix.

<details> 
<summary>sid-base-template.yaml</summary>
  
```
AWSTemplateFormatVersion: 2010-09-09
Description: Storage Immersion Day - foundational shared resources

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: Lab Modules
        Parameters: 
            - includeSecurityLab
            - includePerfLab
            - includeDataSyncLab
            - includeBackupLab
    ParameterLabels:
      includeSecurityLab:
        default: Deploy S3 Security Lab?
      includePerfLab:
        default: Deploy Storage Performance Lab?
      includeDataSyncLab:
        default: Deploy Data Migration Lab?
      includeBackupLab:
        default: Deploy Backup Lab?
      AssetsBucketName:
        default: S3 bucket with CloudFormation templates
      AssetsBucketPrefix:
        default: Prefix to CloudFormation templates

Parameters:
  includeSecurityLab:
    Description: Deploys the resources for the S3 Security Best Practices Lab
    Type: String
    AllowedValues: [true, false]
  includePerfLab:
    Description: Deploys the resources for the Storage Performance Lab
    Type: String
    AllowedValues: [true, false]
  includeDataSyncLab:
    Description: Deploys the resources for the Migrating Data to AWS Lab
    Type: String
    AllowedValues: [true, false]
  includeBackupLab:
    Description: Deploys the resources for the AWS Backup Lab
    Type: String
    AllowedValues: [true, false]
  AssetsBucketName:
    Description: S3 bucket with CloudFormation templates
    Type: String
  AssetsBucketPrefix: 
    Description: Prefix to CloudFormation template location (must end in /)
    Type: String

Conditions:
  includeShared: !Or [!Equals [!Ref includeSecurityLab, true], !Equals [!Ref includePerfLab, true]]
  includeSecurity: !Equals [!Ref includeSecurityLab, true]
  includePerf: !Equals [!Ref includePerfLab, true]
  includeDataSync: !Equals [!Ref includeDataSyncLab, true]
  includeBackup: !Equals [!Ref includeBackupLab, true]

Resources:
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: 10.11.12.0/24
      EnableDnsSupport: true
      EnableDnsHostnames: true
      InstanceTenancy: default
      Tags:
        - Key: Name
          Value: 'SID-vpc'
  Subnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.11.12.0/25
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select 
        - 0
        - !GetAZs 
          Ref: 'AWS::Region'
      Tags:
        - Key: Name
          Value: 'SID-subnet1'
  Subnet2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.11.12.128/25
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select 
        - 1
        - !GetAZs 
          Ref: 'AWS::Region'
      Tags:
        - Key: Name
          Value: 'SID-subnet2'
  InternetGateway:
    Type: 'AWS::EC2::InternetGateway'
    Properties:
      Tags:
        - Key: Name
          Value: 'SID-igw'
  AttachGateway:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
  RouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: 'SID-routes'
  Subnet1RouteAssociaton:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref Subnet1
      RouteTableId: !Ref RouteTable
  Subnet2RouteAssociaton:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref Subnet2
      RouteTableId: !Ref RouteTable
  RoutetoInternet:
    Type: 'AWS::EC2::Route'
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref RouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  SecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: SSH Access
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: 'SID-ssh-sg'
  SecurityGroupIngress:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref SecurityGroup
      IpProtocol: tcp
      ToPort: 2049
      FromPort: 2049
      SourceSecurityGroupId: !Ref SecurityGroup
  SshUsWest2:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: SSH from EC2 instance connect us-west-2
      GroupId: !Ref SecurityGroup
      IpProtocol: tcp
      ToPort: 22
      FromPort: 22
      CidrIp: 18.237.140.160/29

  # Shared resources for security and performance labs
  # We use the GUID from the ARN of the stack ID to generate
  # a unique bucket name
  Bucket:
    Type: 'AWS::S3::Bucket'
    Condition: includeShared
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      BucketName: !Join
      - "-"
      - - "sid-security"
        - !Select
          - 2
          - !Split
            - "/"
            - !Ref "AWS::StackId"

  S3AccessRole:
    Type: 'AWS::IAM::Role'
    Condition: includeShared
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ec2.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
  S3AccessPolicies:
    Type: 'AWS::IAM::Policy'
    Condition: includeShared
    Properties:
      PolicyName: admin
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action:
              - 'ec2:TerminateInstances'
              - 's3:*'
            Resource: '*'
      Roles:
        - !Ref S3AccessRole
  InstanceProfile:
    Type: 'AWS::IAM::InstanceProfile'
    Condition: includeShared
    Properties:
      Path: /
      Roles:
        - !Ref S3AccessRole
  
  # S3 security lab resources
  SecurityStack:
    Type: AWS::CloudFormation::Stack
    Condition: includeSecurity
    DependsOn: InternetGateway
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${AssetsBucketName}/${AssetsBucketPrefix}sid-s3-security-lab.yaml'
      TimeoutInMinutes: 10
      Parameters:
        # userId: !Ref userId
        Bucket: !Ref Bucket
        Subnet1: !Ref Subnet1
        Subnet2: !Ref Subnet2
        SecurityGroup: !Ref SecurityGroup

  # Storage performance lab resources
  PerfStack:
    Type: AWS::CloudFormation::Stack
    Condition: includePerf
    DependsOn: InternetGateway
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${AssetsBucketName}/${AssetsBucketPrefix}sid-performance-lab.yaml'
      TimeoutInMinutes: 10
      Parameters:
        # userId: !Ref userId
        Bucket: !Ref Bucket
        Subnet1: !Ref Subnet1
        Subnet2: !Ref Subnet2
        SecurityGroup: !Ref SecurityGroup
  
  # DataSync lab resources
  DataMigrationStack:
    Type: AWS::CloudFormation::Stack
    Condition: includeDataSync
    DependsOn: InternetGateway
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${AssetsBucketName}/${AssetsBucketPrefix}sid-datamigration-onprem.yaml'
      TimeoutInMinutes: 10
      Parameters:
        Subnet1: !Ref Subnet1
        Subnet2: !Ref Subnet2
        SecurityGroup: !Ref SecurityGroup

  # Backup lab resources
  BackupStack:
    Type: AWS::CloudFormation::Stack
    Condition: includeBackup
    DependsOn: InternetGateway
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${AssetsBucketName}/${AssetsBucketPrefix}sid-backup-lab.yaml'
      TimeoutInMinutes: 60
      Parameters:
        Subnet1: !Ref Subnet1
        Subnet2: !Ref Subnet2
        VPC: !Ref VPC
        SecurityGroup: !Ref SecurityGroup

Outputs:
  S3BucketName:
    Condition: includeShared
    Description: S3 Bucket Name
    Value: !Ref Bucket
  SecurityLabInstance:
    Condition: includeSecurity
    Description: S3 Security Lab Instance Public DNS Name
    Value: !GetAtt SecurityStack.Outputs.SecurityLabInstance
  PerformanceLabInstance:
    Condition: includePerf
    Description: Performance Lab Instance Public DNS Name
    Value: !GetAtt PerfStack.Outputs.PerfLabInstance
  NfsServerPrivateIP:
    Condition: includeDataSync
    Description: NFS Server Private IP Address
    Value: !GetAtt DataMigrationStack.Outputs.NfsServerPrivateIP
  AppServerPrivateIP:
    Condition: includeDataSync
    Description: Application Server Private IP Address
    Value: !GetAtt DataMigrationStack.Outputs.AppServerPrivateIP
  DataSyncAgentPublicIP:
    Condition: includeDataSync
    Description: DataSync Agent Public IP Address
    Value: !GetAtt DataMigrationStack.Outputs.DataSyncAgentPublicIP
  FileGatewayPublicIP:
    Condition: includeDataSync
    Description: File Gateway Public IP Address
    Value: !GetAtt DataMigrationStack.Outputs.FileGatewayPublicIP
```
</details>

<details> 
<summary>sid-s3-security-lab.yaml</summary>
  
```
AWSTemplateFormatVersion: 2010-09-09
Description: Storage Immersion Day - S3 Security Lab resources

Parameters:
  Bucket:
    Description: Name of the S3 Bucket to use for the lab
    Type: String
    AllowedPattern: '[a-zA-Z][a-zA-Z0-9-]*'
    ConstraintDescription: >
      The bucket must begin with a letter and contain only alphanumeric characters or hyphens.
    MinLength: 1
    MaxLength: 64
  Subnet1:
    Description: Subnet ID to use for the EC2 resources
    Type: 'AWS::EC2::Subnet::Id'
  Subnet2:
    Description: Subnet ID to use for the EC2 resources
    Type: 'AWS::EC2::Subnet::Id'
  SecurityGroup:
    Description: Security Group ID to use for the EC2 resources
    Type: String
  LinuxAmi:
    Type : 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'

Resources:
  SidSecurityS3Group:
    Type: 'AWS::IAM::Group'
    Properties:
      GroupName: 'SID-s3-security'
  AddUserToS3Group:
    Type: 'AWS::IAM::UserToGroupAddition'
    Properties:
      GroupName: !Ref SidSecurityS3Group
      Users:
        - !Ref SidSecurityUser1
        - !Ref SidSecurityUser2
  SidSecurityS3Policy:
    Type: 'AWS::IAM::Policy'
    Properties:
      PolicyName: 'SID-s3-access'
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action:
              - 's3:*'
            Resource: '*'
      Groups:
        - !Ref SidSecurityS3Group
  SidSecurityUser1:
    Type: 'AWS::IAM::User'
    Properties:
      UserName: 'SID-lab-user1'
  SidSecurityUser2:
    Type: 'AWS::IAM::User'
    Properties:
      UserName: 'SID-lab-user2'
  SidSecAdminInstance:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: !Ref LinuxAmi
      InstanceType: m5.large
      SubnetId: !Ref Subnet2
      Tags:
        - Key: Name
          Value: 'SID-security-admin'
      SecurityGroupIds:
        - !Ref SecurityGroup
      IamInstanceProfile: !Ref SidInstanceProfile
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            DeleteOnTermination: true
            VolumeSize: 10
      UserData: !Base64 
        'Fn::Join':
          - ''
          - - |
              #!/bin/bash -xe
            - |
              sudo yum update -y
            - >
              AZ=`curl -s
              http://169.254.169.254/latest/meta-data/placement/availability-zone`
            - |
              REGION=${AZ::-1}
            - BUCKET01=
            - !Ref Bucket
            - |+

            - |
              echo 'AdminInstance' | sudo tee -a  /proc/sys/kernel/hostname
            - |
              dd if=/dev/zero of=/tmp/output  bs=1M  count=1
            - >
              aws s3api put-object --bucket $BUCKET01 --key app1/file1 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app1/file2 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app1/file3 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app1/file4 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app1/file5 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app2/file1 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app2/file2 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app2/file3 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app2/file4 --body
              /tmp/output
            - >
              aws s3api put-object --bucket $BUCKET01 --key app2/file5 --body
              /tmp/output
            - |
              sleep 2
            - >
              aws ec2 terminate-instances --instance-ids $(curl -s
              http://169.254.169.254/latest/meta-data/instance-id) --region
              $REGION
  SidSecWorkshopInstance:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: !Ref LinuxAmi
      InstanceType: m5.large
      SubnetId: !Ref Subnet2
      Tags:
        - Key: Name
          Value: 'SID-security-instance'
      SecurityGroupIds:
        - !Ref SecurityGroup
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            DeleteOnTermination: true
            VolumeSize: 10
      UserData: !Base64 
        'Fn::Join':
          - ''
          - - |
              #!/bin/bash -xe
            - |
              sudo yum update -y
            - |
              echo 'storage-workshop' | sudo tee -a  /proc/sys/kernel/hostname
            - |
              echo '#!/bin/bash' > /etc/profile.d/script.sh
            - !Join 
              - ''
              - - echo export bucket=
                - !Ref Bucket
                - |2
                   >> /etc/profile.d/script.sh

  # Instance role and profile
  SidS3AccessRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ec2.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
  SidS3AccessPolicies:
    Type: 'AWS::IAM::Policy'
    Properties:
      PolicyName: admin
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action:
              - 'ec2:TerminateInstances'
              - 's3:*'
            Resource: '*'
      Roles:
        - !Ref SidS3AccessRole
  SidInstanceProfile:
    Type: 'AWS::IAM::InstanceProfile'
    Properties:
      Path: /
      Roles:
        - !Ref SidS3AccessRole

Outputs:
  SecurityLabInstance:
    Value: !GetAtt SidSecWorkshopInstance.PublicDnsName
```
</details>

![assets-folder](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/80074dce-d14f-4ff8-a899-2d4545818b77)

### Launch the templates

Now, we're deploying the templates in the same region by creating a stack called `SID`, short for Storage Immersion Day. Enter the S3 URL of the `base template` in the bucket just created.

![stacksid](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/4444ab70-3104-4420-ac4f-98a6a1758c16)

### Secure Network Access
The CloudFormation templates that are deployed create multiple security groups. To gain access to resources that are provisioned, you will need to edit the security group inbound rules.
Select the security group with the description `SSH Access` and add in a rule that allow HTTP access from your local workstation.

![http-access](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/73b63515-25ef-47e4-9ed6-a33b034830ea)

### Cloud9 Setup
Create a new Cloud9 environment in the Oregon region that will run on a new EC2 instance in the `SID-vpc` vpc and accessed through SSH. 

![cloud9-setip](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/cd01d348-8964-45eb-9dfd-78c00a9a4aeb)

Open the new environment and create a file named `Workshop Info` with the content below:
```

Immersion Day Info:

https://catalog.us-east-1.prod.workshops.aws/workshops/74237958-77c8-4e7f-a02f-ae201a04d759/en-US

ssh-rsa public-key: 

S3 Security Best Practices

SSH to the S3 security instance:
ssh -i "<your key-pair>" "ec2-user@<your EC2 hostname>.compute.amazonaws.com"

Access Keys for IAM Users:

[user1]
aws_access_key_id =
aws_secret_access_key =

[user2]
aws_access_key_id =
aws_secret_access_key =


Amazon S3 Bucket Information:
Bucket name:
Bucket ARN Name:


VPC for the workshop:
VPC:
VPC Endpoint ID:

Storage Performance Lab


Migrating Data to AWS

AppServerPrivateIP:

DataSyncAgentPublicIP:

FileGatewayPublicIP:

NfsServerPrivateIP:

S3BucketName:

BucketRoleForDataSync:


AWS Backup Exercise
```
![file-info](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/fa2367c6-4706-43ab-b1d3-20e27c52533d)

Now that our environment is ready we need to add our Key-pair for SSH access to our Linux hosts. Create one from the EC2 management console and upload the `.pem` file into our Cloud9
environment.

![key-upload](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/45bbf69b-c64a-4fe5-b22a-f8b0af855416)

Now we have to create the public key that will be applied to the `.ssh/authorized_keys` file on the EC2 Linux hosts.
From the Cloud9 command prompt, use the following command to set the permissions of your private key file so that only you can read it.
```bash
chmod 400 cloud9-key.pem
```
Now let's generate the public public key with this command:
```bash
ssh-keygen -y -f cloud9-key.pem
```
Copy the returned ssh-rsa key. Click on your Workshop Info file and paste in it this key  so that it can be referenced for later use.
Connect to the `SID-security-instance` via the EC2 Instance Connect. From the terminal, edit the `.ssh/authorized_keys` to add the ssh-rsa key from the info file. 
For our Cloud9 instance to be able to connect via ssh, we have to add the security group that is associcated with that instance as a source in a new SSH rule in the `SID-ssh-sg` security group.   

Connect to `SID-security-instance` via SSH Client. Copy the ssh command for Public DNS to your clipboard and paste it into your info file in the Cloud9 environment.  Update the `id_rsa` key pair with your own key pair name that was downloaded in a previous step.
Cut and paste this ssh command into your terminal window below in order to connect to the EC2 instance.

![ec2-from9](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/ca544b23-5a22-4fa7-831b-68dcec953f04)






