# EC2 Linux Hands on Lab
## Overview 
The goal of this lab is to deploy an EC2 Linux instance in the default VPC and connect to it.

![amazon-ec2-architecture](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/ddaa9222-055d-4470-8176-8fdd6a8eabe4)

## Create a new Key Pair
We will start by creating a SSH keypair to be used to connect to the EC2 instance. The key pair type is RSA and the available private key file formats are `.ppk` and `.pem`. Since I'm on a Windows, I will choose the `.ppk` format to use it with Putty.

![keypair](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/dcdc6bb2-5d7d-404b-9ce8-446f1886df20)

## Launch a Web Server Instance
Launch an EC2 instance with the name `Web server for IMD` and AMI type `Amazon Linux 2023` and instance type `t2-micro`. Choose the SSH keypair that was created for this lab. The instance will be deployed in the default VPC in a random availabilty zone. A public IP address should be auto-assigned for the instance to be reachable from the internet.
Create a security group with the name and description of `Immersion Day - Web Server`, and add a rule that allow HTTP (TCP 80) traffic to and from the instance. Specify `My IP`as the source type. 

![sg-ec2](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/29f9ff24-4cb6-48c8-96d9-3cb446eb201a)

In the advanced details tab, select `V2 only (token required)` from the `Meta Data version` dropdown, and then add the following script in the `User data`.


```
#!/bin/sh
​
#Install a LAMP stack
dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel
dnf install -y mariadb105-server
dnf install -y httpd php-mbstring
​
#Start the web server
chkconfig httpd on
systemctl start httpd
​
#Install the web pages for our lab
if [ ! -f /var/www/html/immersion-day-app-php7.zip ]; then
   cd /var/www/html
   wget -O 'immersion-day-app-php7.zip' 'https://static.us-east-1.prod.workshops.aws/public/54819b21-675a-42ee-80fc-de26241a3f44/assets/immersion-day-app-php7.zip'
   unzip immersion-day-app-php7.zip
fi
​
#Install the AWS SDK for PHP
if [ ! -f /var/www/html/aws.zip ]; then
   cd /var/www/html
   mkdir vendor
   cd vendor
   wget https://docs.aws.amazon.com/aws-sdk-php/v3/download/aws.zip
   unzip aws.zip
fi
​
# Update existing packages
dnf update -y
```
Browse to the the public DNS name of the instance.
![pagewebserver](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/ccd5f874-b925-429c-9262-6df828607e42)

## Connect to your Linux instance using Session Manager
SSH keys are not needed while using Session Manager. However, we need to create an IAM instance profile for it and attach it to the EC2 instance. To do so, we will first create an IAM role.
Choose `AWS service` as the trusted entity type, select `EC2`as the service that will use this role and attach the ` AmazonSSMManagedInstanceCore` permission policy. Finally, name the role `SSMInstanceProfile`.
Go back to the EC2 console and attach this role to the web server EC2 instance.

![iam-role](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/05f5b4ea-3290-4b8f-a7f6-2cea0af0423d)

You can now connect to the instance via Session Manager.

![ssm](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/cb47aa05-5872-47a2-88cf-b22124e02429)

## Connect to EC2 Instance using PuTTy
We can also connect to the EC2 Instance from the Putty terminal. Just uplaod the `.ppk` file in `Credentials` under the `Auth` tab in Putty then connect to the public IP address of the web server instance and login as `ec2-user`.

![ec2putty](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/186848d1-4aec-44f2-8506-ff6f6e923e5d)

## Clean Up
At the end of the lab, terminate the EC2 instance and delete the security group associated to it.
