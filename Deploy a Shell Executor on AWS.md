## Deploy a Shell Executor on AWS
=======================================


Deploying a Shell Executor GitLab Runner on AWS.

It ll be setting up a GitLab Runner instance that uses the Shell executor (i.e., runs CI/CD jobs directly on the host shell without Docker or Kubernetes). 

Here's a step-by-step guide to do this using a basic EC2 instance on AWS :-


#### 1. Launch EC2 Instance

Choose Ubuntu, or your preferred Linux OS.

key-pair > test1

sg > mysg(all-traffic)

Launch the instance.


#### 2. Install GitLab Runner

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

#### 3. Register GitLab Runner with Shell Executor

Go to gitlab  > create a project (ex- my-first-proj)

select project > CI/CD > Runners > New project runner  >  Tags = shell,aws   >  Create runner

run it

```
gitlab-runner register  --url https://gitlab.com  --token glrt-ksYeA31tJ_G-7kQbw7Ld1286MQpwOjE1Z2VkZgp0OjMKdTpmcWVxOBg.01.1j0dd6wd2
```

Enter an executor :- shell

Check registered runners

```
sudo gitlab-runner list
```


#### 4. Configure and Start the Runner

```
sudo systemctl enable gitlab-runner
sudo systemctl start gitlab-runner
```

#### 5. Check status

```
sudo systemctl status gitlab-runner
```

#### 6. Test Runner in GitLab

create `.gitlab-ci.yml`

```
test_shell:
  tags:
    - shell
  script:
    - echo "Running on AWS Shell Runner"
    - uname -a

```

Check your pipeline logs in GitLab CI/CD → Pipelines.

=========END===================
