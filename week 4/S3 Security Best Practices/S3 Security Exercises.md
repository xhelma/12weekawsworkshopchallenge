# Prepare your Lab
Before we can start the exercises for this lab we need to ensure the AWS CLI is using the proper credentials.  

We need to create access keys for `SID-lab-user1` and `SID-lab-user2` from the IAM
console. Fill in the following credentials file template with the access key IDs and the secret access key IDs of the users:

```
[user1]
aws_access_key_id =
aws_secret_access_key =
[user2]
aws_access_key_id =
aws_secret_access_key =
```
Connect to the `SID-security-instance` EC2 instance via the EC2 Instance Connect. In the CLI for the instance, run the following commands to setup the AWS CLI:
```
aws configure
```
Leave Access Key and Secret Key blank by pressing return. Set the region to `us-west-2`.

![ec2-connect](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/f6d5c01e-2dc8-4021-aafe-5d983dd22cf8)

Create a credentials file to be used by the AWS CLI using the following command. This will allow you to easily switch between two different IAM users.
```
cd ~/.aws
vi credentials
```
Insert the credentials file already filled in and save it.  

Copy the name of the S3 bucket deployed by the CloudFormation template in the setup step that has the format `sid-security-xxxxxxx` and save it for later use. By default, public access to 
this bucket is blocked. For this lab, we'll need to enable it alongside the ACLs. 

# S3 Security Exercises
The purpose of the exercises is to demonstrate the use of bucket policies, access points, and blocking public access to secure S3 buckets.

## Require HTTPS
In this exercise we will create a S3 Bucket Policy that requires connections to use HTTPS.  

To do so, copy the bucket policy below, and paste into the Bucket Policy Editor. Replace `BUCKET_NAME` with the bucket name you previously saved.
```
{
"Statement": [
{
   "Action": "s3:*",
   "Effect": "Deny",
   "Principal": "*",
   "Resource": "arn:aws:s3:::BUCKET_NAME/*",
   "Condition": {
       "Bool": {
        "aws:SecureTransport": false
        }
    }
    }
  ]
}
```
![s3-policy](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/654fd53b-dbb4-4153-b548-70e3371cf810)

From the EC2 terminal, run the following command:
```
aws s3api head-object --key app1/file1 --endpoint-url http://s3.amazonaws.com --profile user1 --bucket ${bucket}
```
A 403 error is returned since the endpoint-url is HTTP.
![403-error](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/4559b9fa-9367-44fd-a0ef-1b8d4180efa9)

Now run the following command:
```
aws s3api --endpoint-url https://s3.amazonaws.com --profile user1 head-object --key app1/file1 --bucket ${bucket}
```
The command succeeds because the endpoint-url uses HTTPS as required by the bucket policy.
![https-success](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/7a18015a-359e-4436-a110-1957f1e2bb5a)

## Require SSE-S3 Encryption
In this exercise we will create a S3 Bucket Policy that requires data at rest encryption.  

To do so, copy the bucket policy below, and paste into the Bucket Policy Editor. Replace `BUCKET_NAME` with the bucket name you previously saved.
```
{
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::BUCKET_NAME/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "AES256"
                }
            }
        }
    ]
}
```
![policy2](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/4cb85f5f-6f21-4777-99dd-f3f4e9981477)

From the EC2 terminal, run the following command to create the `textfile` object:
```
cd ~  
echo "123456789abcdefg" > textfile  
```
Run the following command in your SSH session to put it to your bucket.
```
aws s3api put-object --key text01 --body textfile --profile user1 --bucket ${bucket}    
```
The request should fail, as the object is not encrypted using SSE-S3 encryption as required by your bucket policy.
![call fail](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/e1ce02f3-3c8c-43d4-96f1-d7bc88d48614)

Now run the following command using SSE-S3 encryption:
```
aws s3api put-object --key text01 --body textfile --server-side-encryption AES256 --profile user1 --bucket ${bucket}  
```
The command succeeds because the PUT used SSE-S3.

![call-succes](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/15f48b92-7220-4f37-be84-d8e098b58dbf)

## Block Public ACLs
In this exercise we will demonstrate the use of block public ACLs.

Now, copy the bucket policy below, and paste into the Bucket Policy Editor. Replace `BUCKET_NAME` with the bucket name.
```
{
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::BUCKET_NAME/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "private"
                }
            }
        }
    ]
}
```
From the EC2 terminal, run the following command:
```
aws s3api put-object --key text01 --body textfile --profile user1 --bucket ${bucket}  
```
![policy1-succ1](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/7ecc45ee-4e7a-47e4-a4ce-503bb6998931)

The request should succeed since the default for an object ACL is private.
 
Now run the following command:
```
aws s3api put-object --key text01 --body textfile --acl public-read --profile user1 --bucket ${bucket}   
```
![policy3-succ2](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/648dee9b-c608-46da-b697-5e880f833e39)

This command also succeed since the current bucket policy allows ACLs that are private but doesn't DENY anything. It is important to write policies that prevent actions,
not allow it when trying to restrict actions against a bucket. The current bucket policy also allows Public access to the bucket unintentionally due to the principal being a wildcard.
Replace the existing policy with the following policy. Notice this is a deny policy.

```
{
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": [
                    "s3:PutObject",
                    "s3:PutObjectAcl"
                    ],
            "Resource": "arn:aws:s3:::BUCKET_NAME/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": [
                           "public-read",
                           "public-read-write",
                           "authenticated-read"
                    ]
                }
            }
        }
    ]
}   
```

From the EC2 terminal, run the following command:
```
aws s3api put-object --key text01 --body textfile --profile user1 --bucket ${bucket}  
```
![policy1-succ1](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/91f053eb-3b5e-47ee-a5e1-2df978bc9c2d)

The request should succeed since the default for an object ACL is private.

Now run the following command:
```
aws s3api put-object --key text01 --body textfile --acl public-read --profile user1 --bucket ${bucket}   
```
The request fails as the bucket policy now restricts public-read ACL.

![fail3](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/e1492280-c770-4328-acaa-739cfa44f627)

## Configure S3 Block Public Access
In this exercise we will configure S3 Block Public Access. Delete the existing bucket policy and select `Block public access to buckets and objects granted through new access control lists (ACLs)`
under the block access section.

![block1](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/df43443d-4acf-4cda-9171-37c4abf50aa8)

From the EC2 terminal, run the following command:
```
aws s3api put-object --key text01 --body textfile --profile user1 --bucket ${bucket}  
```
![policy1-succ1](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/91f053eb-3b5e-47ee-a5e1-2df978bc9c2d)

The request should succeed since the default for an object ACL is private.

Now run the following command:
```
aws s3api put-object --key text01 --body textfile --acl public-read --profile user1 --bucket ${bucket}   
```
The requests fails as the bucket policy restricts the public-read ACL.
![fail3](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/e1492280-c770-4328-acaa-739cfa44f627)

Don't forget to `Unselect Block public access to buckets and objects granted through new access control lists (ACLs)` before moving on to the next exercise.

## Restrict Access to a S3 VPC Endpoint
In this exercise we will configure a S3 VPC Endpoint and a bucket policy to limit access to only requests that pass through the VPC Endpoint. 
This is an easy way to limit access to only clients in your VPC.  
From the VPC console, create an endpoint with the name `sid-vpce-s3` and choose S3 as the service and `Gateway` as type. Select `SID-vpc` as VPC.

![endpoint-vpc](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/24feac30-7b99-4f0d-828c-01a7c8b28559)

Copy the bucket policy below, and paste into the Bucket Policy Editor. Replace `BUCKET_NAME` with the bucket name and `VPC_ENDPOINT_ID` with the VPC endpoint ID generated.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "s3:*",
            "Effect": "Deny",
            "Resource": "arn:aws:s3:::BUCKET_NAME/*",
            "Condition": {
                "StringNotEquals": {
                    "aws:sourceVpce": "VPC_ENDPOINT_ID"
                }
            },
            "Principal": "*"
        }
    ]
}
```
From the SSH session run the following command:
```
aws s3api head-object --key app1/file1 --profile user1 --bucket ${bucket}
```
which returns this:
![policy4-403](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/50889c95-c6da-464d-a3a5-8d36194b6049)
The HeadObject request failed because there is no route table associated with the VPC Endpoint. To remediate this, select the Route Table ID associated with `SID-routes` to attach it 
to the endpoint. We then try the command a second time:

![policy4succ](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/3621f240-08f3-4574-bee6-cf341d192b00)

And voilà! The request succeeds because a route is associated to the VPC Endpoint.

## Use AWS Config Rules to Detect a Public Bucket
In this exercise, the AWS `Config` service is used.  
Create a new rule and attach to it the predefined rule `s3-bucket-public-read-prohibited`. Navigate to our bucket and edit the access control list (ACL) by selecting `List` and `Read`
under `Everyone(public access)`.  

Go back to the config service and check the `s3-bucket-public-read-prohibited` rule.   
![config-aws](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/cc3a053a-8a1d-4698-b190-b5405ce0c4a3)

The non-compliant S3 bucket shows up in the ressources in scope table.

## Use Amazon Access Analyzer for S3
Head over to the IAM console and create an analyser with the default settings.  
Go back to S3 console, specifically `Access analyzer for S3` section. Notice that the bucket allows access to anyone on the internet. This is due to the changes made in the last
exercise.

![s3-analyzer](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/a7e9927d-9d83-49cd-8a1d-0ef1a94bb5f4)

Navigate to our bucket and edit the access control list (ACL) by deselecting `List` and `Read` under `Everyone(public access)`.  

From the IAM Access analyser, we can see that the status of the
bucket has changed to resolved.

![iam-ana](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/e64f5291-fd7a-40a0-a310-6337f79c5b58)




