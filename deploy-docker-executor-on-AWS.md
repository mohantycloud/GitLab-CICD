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
sudo apt install gitlab-runner
```

