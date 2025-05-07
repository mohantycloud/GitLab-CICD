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

#### 2. Install GitLab Runner

```
sudo curl -L https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64 -o /usr/local/bin/gitlab-runner

```


```
sudo chmod +x /usr/local/bin/gitlab-runner
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

```
mkdir -p /etc/gitlab-runner/custom-executor
cd /etc/gitlab-runner/custom-executor
```

`vi prepare.sh`

```
cat <<'EOF' > prepare
#!/bin/bash
TASK_ARN=$(aws ecs run-task \
  --cluster gitlab-runner-fargate \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxxx],securityGroups=[sg-xxxxxx],assignPublicIp=ENABLED}" \
  --task-definition gitlab-runner-task \
  --query 'tasks[0].taskArn' \
  --output text)

echo $TASK_ARN > /tmp/gitlab-task-arn
EOF
```

`vi run.sh`

```
cat <<'EOF' > run
#!/bin/bash
echo "Job running in Fargate... waiting"
sleep 60
EOF
```


`vi cleanup.sh`

```
cat <<'EOF' > cleanup
#!/bin/bash
TASK_ARN=$(cat /tmp/gitlab-task-arn)
aws ecs stop-task --cluster gitlab-runner-fargate --task $TASK_ARN
EOF
```

```
chmod +x prepare run cleanup
```

#### 7. Configure config.toml

```
nano /etc/gitlab-runner/config.toml
```


```
[[runners]]
  name = "fargate-runner"
  url = "https://gitlab.com/"
  token = "REPLACE_WITH_YOUR_TOKEN"
  executor = "custom"
  [runners.custom]
    config_exec = "/etc/gitlab-runner/custom-executor/prepare"
    run_exec = "/etc/gitlab-runner/custom-executor/run"
    cleanup_exec = "/etc/gitlab-runner/custom-executor/cleanup"
```


#### 8. Start the Runner

```
gitlab-runner run
```


#### 9. Test Your GitLab Runner on Fargate



`.gitlab-ci.yml`


```
stages:
  - test

fargate_test_job:
  stage: test
  tags:
    - fargate   # <- must match your GitLab Runner tag
  script:
    - echo "✅ Hello from GitLab CI/CD running on AWS Fargate!"
    - uname -a
    - sleep 10
```

Your Job Runs in AWS Fargate!
