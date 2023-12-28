# Auto Scaling and Load Balancing

## Create a Load Balancer

1. First, create two other subnets in the same VPC to host the other 2 instances. Get the Availability Zone (AZ) where the current subnet exists. Then, ensure that the other two subnets are in different AZs. Note down the subnet IDs produced from the following commands.
    ```
    INSTID=i-xxxxxxxxxxxxxxxxx
    VPCID=$(aws ec2 describe-instances --instance-ids $INSTID --query "Reservations[*].Instances[*].VpcId" --output text)
    SUBNETID=$(aws ec2 describe-instances --instance-ids $INSTID --query "Reservations[*].Instances[*].SubnetId" --output text)
    echo $(aws ec2 describe-subnets --subnet-ids $SUBNETID --query "Subnets[*].AvailabilityZone" --output text)
    aws ec2 create-subnet --vpc-id $VPCID --cidr-block $CIDRBLK2 --availability-zone us-east-1a
    aws ec2 create-subnet --vpc-id $VPCID --cidr-block $CIDRBLK3 --availability-zone us-east-1b
    ```
    ```
    SUBNETID2=subnet-xxxxxxxxxxxxxxxxx
    SUBNETID3=subnet-yyyyyyyyyyyyyyyyy
    ```
    **NOTE**: Ensure that the route table and network ACL associated with all the subnets are the same.

2. Create target group. Note down the target group ARN output by the command.
    ```
    TARGETGRP=LlmGpt4allFastapiTargetGrp
    aws elbv2 create-target-group --name $TARGETGRP --protocol HTTP --port 8080 --vpc-id $VPCID
    ```
    ```
    TARGETGRPARN=arn:aws:elasticloadbalancing:us-east-1:<aws_account_id>:targetgroup/LlmGpt4allFastapiTargetGrp/xxxxxxxxxxxxxxxx
    ```

3. Create load balancer. Note down the DNS name and ARN of the load balancer.
    ```
    LBNAME=LlmGpt4allFastapiLB
    aws elbv2 create-load-balancer --name $LBNAME --subnets $SUBNETID $SUBNETID2 $SUBNETID3 --type application
    ```
    ```
    LBARN=arn:aws:elasticloadbalancing:us-east-1:<aws_account_id>:loadbalancer/app/LlmGpt4allFastapiLB/xxxxxxxxxxxxxxxx
    LBDNS=LlmGpt4allFastapiLB-xxxxxxxxx.us-east-1.elb.amazonaws.com
    ```

4. Create a listener for incoming requests.
    ```
    aws elbv2 create-listener --load-balancer-arn $LBARN --protocol HTTP --port 8080 --default-actions Type=forward,TargetGroupArn=$TARGETGRPARN
    ```

## Set Up an Auto Scaling Group

1. Get the instance id of the EC2 instance set up in the previous guide, and create a custom AMI based on this instance. Note down the AMI id output from this command.
    ```
    aws ec2 create-image --instance-id $INSTID --name "LlmGpt4allFastapiAMI" --description "AMI with container serving GPT4All LLM via FastAPI."
    ```
    ```
    AMIID=ami-yyyyyyyyyyyyyyyyy
    ```

2. Create launch configuration.
    ```
    LAUNCHCFG=LlmGpt4allFastapiLC
    INSTTYPE=c5a.4xlarge
    KEYNAME=LlmGpt4allFastapiKeyPair
    GRPNAME=LlmGpt4allFastapiSecurityGroup
    aws autoscaling create-launch-configuration --launch-configuration-name $LAUNCHCFG --image-id $AMIID --instance-type $INSTTYPE --key-name $KEYNAME --security-groups $GRPNAME
    ```

3. Create an EC2 Auto Scaling Group (ASG).
    ```
    ASGNAME=LlmGpt4allFastapiASG
    MINCAP=0
    MAXCAP=3
    DESIREDCAP=2
    SUBNETID=$(aws ec2 describe-instances --instance-ids $INSTID --query "Reservations[*].Instances[*].SubnetId" --output text)
    aws autoscaling create-auto-scaling-group --auto-scaling-group-name $ASGNAME --launch-configuration-name $LAUNCHCFG --min-size $MINCAP --max-size $MAXCAP --desired-capacity $DESIREDCAP --vpc-zone-identifier $SUBNETID
    ```

4. Define scale out and scale in policies. Note down the ARNs of the policies from the outputs of the following commands.
    ```
    SOPOL=LlmGpt4allFastapiSOPolicy
    SIPOL=LlmGpt4allFastapiSIPolicy
    aws autoscaling put-scaling-policy --auto-scaling-group-name $ASGNAME --policy-name $SOPOL --scaling-adjustment 1 --adjustment-type ChangeInCapacity
    aws autoscaling put-scaling-policy --auto-scaling-group-name $ASGNAME --policy-name $SIPOL --scaling-adjustment -1 --adjustment-type ChangeInCapacity
    ```
    ```
    SOPOLARN=arn:aws:autoscaling:us-east-1:<aws_account_id>:scalingPolicy:<xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:autoScalingGroupName/LlmGpt4allFastapiASG:policyName/LlmGpt4allFastapiSOPolicy
    SIPOLARN=arn:aws:autoscaling:us-east-1:<aws_account_id>:scalingPolicy:<yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy:autoScalingGroupName/LlmGpt4allFastapiASG:policyName/LlmGpt4allFastapiSIPolicy
    ```

5. Attach load balancer.
    ```
    aws autoscaling attach-load-balancer-target-groups --auto-scaling-group-name $ASGNAME --target-group-arns $TARGETGRPARN
    ```

6. Update capacity as follows.
    ```
    aws autoscaling attach-load-balancer-target-groups --auto-scaling-group-name $ASGNAME --target-group-arns $TARGETGRPARN
    ```
