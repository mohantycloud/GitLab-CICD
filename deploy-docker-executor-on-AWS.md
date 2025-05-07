## Deploy Self-Hosted Runner(Docker Executor) on AWS
==========================================================


1. launch a remote machine(aws ec2)
2. install gitlab-runner in aws ec2
3. install docker in same ec2
4. register runner on gitlab project


#### step-1 :- create ec2 instance

ami > ubuntu

key-pair > test1

sg > mysg(all-traffic)

connect

#### step-2 :- install gitlab-runner

Follow `https://docs.gitlab.com/runner/install/linux-repository/`

```
sudo apt update
```

Add the official GitLab repository

```
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
```

Install the latest version of GitLab Runner

```
sudo apt install gitlab-runner -y
```

#### step-3 :- install docker in same ec2

```
sudo apt update
```

```
sudo apt install -y ca-certificates curl
```

```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```
sudo chmod 666 /var/run/docker.sock
```

```
sudo systemctl start docker
sudo systemctl enable docker
```

```
docker --version
```


#### step-4 :- register runner on gitlab project

Go to gitlab  > create a project (ex- my-first-proj)

select project > CI/CD > Runners > New project runner  >  Tags = build-docker-runner   >  Create runner

run it

```
gitlab-runner register  --url https://gitlab.com  --token glrt-ksYeA31tJ_G-7kQbw7Ld1286MQpwOjE1Z2VkZgp0OjMKdTpmcWVxOBg.01.1j0dd6wd2
```

Manually verify that the runner is available to pick up jobs

```
gitlab-runner run
```

create `.gitlab-ci.yml`

```

build-job:
  stage: build
  tags:
    - build-docker-runner
  script:
    - echo "Running on AWS Docker runner"
```


