# Large Language Model (LLM) on ECS

This guide explains how to create a model serving endpoint for an LLM (think, ChatGPT-like models) by setting up a FastAPI server on ECS. This option is ideal if you'd rather AWS manage the scaling and provisioning of resources instead of hand-managing an EC2 instance.

NOTE: we'll be using the image creating in the [previous guide](./llm-gpt4all-fastapi#package-the-service-into-a-container-image).

## Create an execution role

First, we need to create a role with which ECS will execute tasks.

1. Create a file called `ecs-trust-policy.json` to define the trust policy as follows.
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "ecs-tasks.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }
    ```

2. Create an IAM role. Save the role ARN output by the command.
    ```
    aws iam create-role --role-name EcsExecutionRole --assume-role-policy-document file://ecs-trust-policy.json
    ```
    ```
    ECSEXECROLEARN=arn:aws:iam::<aws_account_id>:role/EcsExecutionRole
    ```

3. Attach the required policy to the role created above.
    ```
    aws iam attach-role-policy --role-name EcsExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
    ```

## Create a subnet and security group

Next, we need to create a subnet within which the ECS containers will live. We also need to create a security group to define network access rules.

1. Create subnet by running the following. Make sure that the subnet CIDR block is not something that's already in use by another resource. Note down the subnet ID output from the command.
    ```
    VPCID=vpc-xxxxxxxxxxxxxxxxx
    SUBNETCIDR=172.31.144.0/20
    AZ=us-east-1f
    aws ec2 create-subnet --vpc-id $VPCID --cidr-block $SUBNETCIDR --availability-zone AZ
    ```
    ```
    ECSSUBNETID=subnet-xxxxxxxxxxxxxxxxx
    ```

2. Create security group. Note down the security group ID output by the command.
    ```
    GRPNAME=LlmGpt4allFastapiEcsSecurityGroup
    aws ec2 create-security-group --group-name $GRPNAME --description "SG for GPT4All LLM deployment on ECS" --vpc-id $VPCID
    ```
    ```
    ECSSGID=sg-xxxxxxxxxxxxxxxxx
    ```

3. Allow list current device (or the set of target devices) to access the ECS containers at the specified port.
    ```
    PUBLICIP=$(curl ipinfo.io/ip)
    aws ec2 authorize-security-group-ingress --group-id $ECSSGID --protocol tcp --port 8080 --cidr $PUBLICIP/32
    ```

## Set Up ECS task in an ECS cluster

Finally, we will set up a cluster and create a task i.e. running containers.

1. Create an ECS cluster.
    ```
    CLUSTERNAME=LlmGpt4allFastapiEcsCluster
    aws ecs create-cluster --cluster-name $CLUSTERNAME
    ```

1. Create a `task-definition.yaml` file with the following content.
    ```
    family: "llm-gpt4all-ecs-server"
    networkMode: "awsvpc"
    requiresCompatibilities:
    - "FARGATE"
    cpu: "16384"
    memory: "65536"
    executionRoleArn: "arn:aws:iam::<aws_account_id>:role/EcsExecutionRole"
    containerDefinitions:
    - name: llm-gpt4all-container
        image: <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/llm-gpt4all-fastapi-demo:latest
        cpu: 16384
        memory: 65536
        portMappings:
        - containerPort: 8080
        essential: true
    ```

2. Register the task definiton created in the above file. Note down the task ARN output by the command.
    ```
    aws ecs register-task-definition --cli-input-yaml file://task-definition.yaml
    ```
    ```
    ECSTASKARN=arn:aws:ecs:us-east-1:<aws_account_id>:task/LlmGpt4allFastapiEcsCluster/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    ```

3. Create a service, using the subnets and security groups created in the [previous section](#create-a-subnet-and-security-group). Note down the service ARN output by the command.
    ```
    SVCNAME=llm-gpt4all-fastapi-svc
    aws ecs create-service \
    --cluster $CLUSTERNAME \
    --service-name $SVCNAME \
    --task-definition llm-gpt4all-ecs-server \
    --desired-count 1 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$ECSSUBNETID],securityGroups=[$ECSSGID],assignPublicIp=ENABLED}"
    ```
    ```
    ECSSVCARN=arn:aws:ecs:us-east-1:<aws_account_id>:service/LlmGpt4allFastapiEcsCluster/llm-gpt4all-fastapi-svc
    ```

4. You're all set! Ping the service from one of the allow-listed devices to see it in action.
```
ENIID=eni-08843015452dd4218
SVCPUBLICIP=$(aws ec2 describe-network-interfaces --network-interface-ids $ENIID --query 'NetworkInterfaces[*].Association.PublicIp' --output text)
```

## Auto Scaling and Load Balancing [WORK IN PROGRESS]

1. set up asg
    ```
    aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/$CLUSTERNAME/$SVCNAME \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 0 \
    --max-capacity 3
    ```

2. add policy
```
    aws application-autoscaling put-scaling-policy \
    --policy-name fastapi-demo-cpu-scaling \
    --service-namespace ecs \
    --resource-id service/$CLUSTERNAME/$SVCNAME \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration "{
        \"PredefinedMetricSpecification\": {
            \"PredefinedMetricType\": \"ECSServiceAverageCPUUtilization\"
        },
        \"TargetValue\": 5.0,
        \"ScaleInCooldown\": 60,
        \"ScaleOutCooldown\": 60
    }"
```
NOTE: The above policy creates CloudWatch alarms with actions associated with them. You don't get granular control over the definition of these alarms from the `application-autoscaling` command. For example, if you want to scale in based on 10 data points in 10 minutes, you cannot set it in the above command.

3. Add load balancer at the front.
