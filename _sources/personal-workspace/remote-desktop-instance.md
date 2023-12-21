# Remote Desktop Instance

This doc walks (for the most part, future me) through setting up a "remote desktop" i.e. VM instance in the cloud to serve as a development / research environment.

## AWS

You can do this via the AWS console, the AWS CLI, or AWS CDK. I prefer the CLI (for a myriad of reasons, which I might write about some day), so the following steps assume the same.

0. Install the AWS CLI if you have not already, by following instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

1. Log in as your AWS account/user by running `aws configure`.

2. Create an EC2 Key Pair if you don't have one already, and set correct permissions on the key file.
```
KEYNAME=DevEnvKeyPair
aws ec2 create-key-pair --key-name $KEYNAME --query 'KeyMaterial' --output text > $KEYNAME.pem
chmod 400 $KEYNAME.pem
```

3. Create a security group for this instance within the VPC in which it lies (You can find ids of all VPCs by running `aws ec2 describe-vpcs`). Note down the id of the security group from the output of the first command. Next, add an ingress rule to allow SSH access from your machine's public IP.
```
GRPNAME=DevEnvSecurityGroup
VPCID=vpc-xxxxxxxxxxxxxxxxx
PUBLICIP=255.255.255.255
aws ec2 create-security-group --group-name $GRPNAME --description "Security Group for my devel environment" --vpc-id $VPCID
aws ec2 authorize-security-group-ingress --group-name $GRPNAME --protocol tcp --port 22 --cidr $PUBLICIP/32
```

4. Launch an EC2 instance with Ubuntu 22.04 AMI. The AMI id used here will get outdated eventually so first check for newer versions of Ubuntu 22.04 AMI [here](https://cloud-images.ubuntu.com/locator/ec2/). Note down the id of the launched instance from the output of the command.
```
AMIID=ami-0fc5d935ebf8bc3bc
SGID=sg-0425c9b1965d7d884
INSTTYPE=c5.2xlarge
aws ec2 run-instances --image-id $AMIID --instance-type $INSTTYPE --count 1 --key-name $KEYNAME --security-group-ids $SGID --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=DevEnv}]' --block-device-mappings '[{"DeviceName": "/dev/sdf", "Ebs": {"VolumeSize": 12, "DeleteOnTermination": false, "VolumeType": "gp2"}}]'
```

5. To SSH into the instance, first get the public IP and public DNS of the instance. Then run the SSH command using these values 
```
INSTID=i-xxxxxxxxxxxxxxxxx
PUBLICDNS=$(aws ec2 describe-instances --instance-ids $INSTID --query "Reservations[*].Instances[*].{PublicDNS:PublicDnsName}" --output text)
ssh -i $KEYNAME.pem ubuntu@$PUBLICDNS
```
