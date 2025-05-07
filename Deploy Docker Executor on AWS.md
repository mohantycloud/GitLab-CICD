## Deploy a Shell Executor on AWS
=======================================


#### 1. Launch EC2 Instance

Choose Ubuntu, or your preferred Linux OS.

key-pair > test1

sg > mysg(all-traffic)

Launch the instance.


#### 2. Install Docker

```
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

Add GitLab Runner to Docker group (for permission)

```
sudo usermod -aG docker gitlab-runner
```


#### 3. Install GitLab Runner

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

Check the installation

```
gitlab-runner --version
```

#### 4. Register GitLab Runner with Shell Executor

Go to gitlab  > create a project (ex- my-second-proj)

select project > CI/CD > Runners > New project runner  >  Tags = docker,aws   >  Create runner

run it

```
gitlab-runner register  --url https://gitlab.com  --token glrt-ksYeA31tJ_G-7kQbw7Ld1286MQpwOjE1Z2VkZgp0OjMKdTpmcWVxOBg.01.1j0dd6wd2
```

Enter an executor :- docker

Check registered runners

```
sudo gitlab-runner list
```

#### 5. Configure and Start the Runner

```
sudo systemctl enable gitlab-runner
sudo systemctl start gitlab-runner
```

#### 6. Check status

```
sudo systemctl status gitlab-runner
```

Verify Docker access (optional test)

```
sudo docker run hello-world
```

#### 7. Test Runner in GitLab

create `.gitlab-ci.yml`

```
stages:
  - test

test_docker:
  stage: test
  tags:
    - docker
  image: python:3.10
  script:
    - echo "Running inside Docker on AWS"
    - python --version

```

Check your pipeline logs in GitLab CI/CD → Pipelines.


=========END===================
