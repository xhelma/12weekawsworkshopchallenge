As a setup, we're going to need to deploy the necessary ressources via AWS CloudFormation.
Let's start by creating a bucket in the `us-west-2` area that will be hosting the CloudFormation templates. Upload to it the `Base Template` and `S3 Best Security Practices` template
under the `assets` prefix.

```
<details> 
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
</details>
```
