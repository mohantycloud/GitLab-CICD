## Step-by-Step CLI Guide
====================================

#### 1. Prepare CLI Tools and Environment

`https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html`


AWS EC2 INSTANCE(UBUNTU)

```
sudo apt update
sudo apt install unzip curl -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

```
aws configure
```

Generate  `Access key`

#### 2. ECS Cluster

```
aws ecs create-cluster --cluster-name gitlab-runner-cluster
```

#### 3. Task Execution Role

Create IAM Role for ECS Tasks

`vi ecs-trust-policy.json`

```

{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "ecs-tasks.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}

```


create-role

```
aws iam create-role \
  --role-name GitLabRunnerFargateRole \
  --assume-role-policy-document file://ecs-trust-policy.json
```



Attach policies

```
aws iam attach-role-policy \
  --role-name GitLabRunnerFargateRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

```
aws iam attach-role-policy \
  --role-name GitLabRunnerFargateRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonECS_FullAccess

```

#### 4. Create ECS Task Definition

`vi task-definition.json`

```

{
  "family": "gitlab-runner-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [{
    "name": "gitlab-job",
    "image": "alpine",
    "entryPoint": ["sh", "-c"],
    "command": ["echo Hello from GitLab Fargate job && sleep 60"],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/gitlab-runner",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }],
  "executionRoleArn": "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/GitLabRunnerFargateRole"
}

```

Register the task

```
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json
```
`press :- q`

Create a CloudWatch log group

```
aws logs create-log-group --log-group-name /ecs/gitlab-runner
```
