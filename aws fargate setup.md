## Step-by-Step CLI Guide
====================================

#### 1. Prepare CLI Tools and Environment

```
sudo apt install awscli -y
```

```
aws configure
```

#### 2. Install GitLab Runner

```
curl -L https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64 \
  -o /usr/local/bin/gitlab-runner
```


```
chmod +x /usr/local/bin/gitlab-runner
```

```
gitlab-runner --version
```

#### 3. Create ECS Cluster


```
aws ecs create-cluster --cluster-name gitlab-runner-fargate
```

Create IAM Role for ECS Tasks

`vi ecs-trust-policy.json`

```
cat <<EOF > ecs-trust-policy.json
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
EOF
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

aws iam attach-role-policy \
  --role-name GitLabRunnerFargateRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerServiceFullAccess
```

#### 4. Create ECS Task Definition

`vi task-definition.json`

```
cat <<EOF > task-definition.json
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
EOF
```

Register the task

```
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json
```

Create a CloudWatch log group

```
aws logs create-log-group --log-group-name /ecs/gitlab-runner
```

#### 5. Register GitLab Runner

Go to gitlab  > create a project (ex- my-awsfargate-proj)

select project > CI/CD > Runners > New project runner  >  Tags = fargate   >  Create runner

run it

```
gitlab-runner register  --url https://gitlab.com  --token glrt-ksYeA31tJ_G-7kQbw7Ld1286MQpwOjE1Z2VkZgp0OjMKdTpmcWVxOBg.01.1j0dd6wd2
```

Enter an executor :- custom

Check registered runners

```
sudo gitlab-runner list
```

#### 6. Setup Custom Executor Scripts
