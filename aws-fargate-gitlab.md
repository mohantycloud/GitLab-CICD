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
        "awslogs-region": "ap-south-1",
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


#### 5. Install GitLab Runner with Custom Executor

```
sudo curl -L https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64 -o /usr/local/bin/gitlab-runner

```


```
sudo chmod +x /usr/local/bin/gitlab-runner
```

```
gitlab-runner --version
```

#### 6. Register Runner with GitLab

Go to gitlab  > create a project (ex- fargate-runner)

select project > CI/CD > Runners > New project runner  >  Tags = skipp it   >  Create runner

run it

```
sudo gitlab-runner register
```

Executor: custom

Name: fargate-runner

URL: https://gitlab.com (or your self-hosted GitLab)

Token: Use the registration token

Check registered runners

```
sudo gitlab-runner list
```



#### 7. Setup Custom Executor Scripts

```
sudo mkdir -p /etc/gitlab-runner/fargate
sudo chown -R $(whoami) /etc/gitlab-runner
cd /etc/gitlab-runner/fargate
```

`vi config.toml`

```
[[runners]]
  name = "fargate-runner"
  url = "https://gitlab.com/"
  token = "glrt-MN5lI9qF5Qm733kZgGUtuW86MQpwOjE1Z243NQp0OjMKdTpmcWVxOBg.01.1j0mxw8rt"
  executor = "custom"
  builds_dir = "/builds"
  shell = "bash"

  [runners.custom]
    prepare_exec = "/etc/gitlab-runner/fargate/prepare"
    run_exec = "/etc/gitlab-runner/fargate/run"
    cleanup_exec = "/etc/gitlab-runner/fargate/cleanup"
```



`vi prepare`


```
#!/bin/bash
set -e

echo "[prepare] Starting ECS task for job: $CI_JOB_ID"

TASK_ARN=$(aws ecs run-task \
  --cluster gitlab-runner-cluster \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-00fdb2415ceeca4ee],securityGroups=[sg-03291c1a2530f9496],assignPublicIp=ENABLED}" \
  --task-definition gitlab-runner-task \
  --overrides '{"containerOverrides": [{"name": "gitlab-job", "environment": [{"name": "CI_JOB_ID", "value": "'"$CI_JOB_ID"'"}]}]}' \
  --query 'tasks[0].taskArn' \
  --output text)

echo "$TASK_ARN" > /tmp/task_arn_"$CI_JOB_ID"

echo "[prepare] Task started: $TASK_ARN"
```


`vi run`


```
#!/bin/bash
set -e

TASK_ARN=$(cat /tmp/task_arn_"$CI_JOB_ID")

echo "[run] Waiting for task $TASK_ARN to finish..."

aws ecs wait tasks-stopped \
  --cluster gitlab-runner-cluster \
  --tasks "$TASK_ARN"

echo "[run] Task $TASK_ARN finished"
```

`vi cleanup`

```
#!/bin/bash
set -e

TASK_ARN_FILE="/tmp/task_arn_$CI_JOB_ID"

if [ -f "$TASK_ARN_FILE" ]; then
  TASK_ARN=$(cat "$TASK_ARN_FILE")
  echo "[cleanup] Cleaning up task: $TASK_ARN"

  # Uncomment below if you want to force stop task
  # aws ecs stop-task --cluster gitlab-runner-cluster --task "$TASK_ARN"

  rm -f "$TASK_ARN_FILE"
else
  echo "[cleanup] No task found for cleanup."
fi
```

