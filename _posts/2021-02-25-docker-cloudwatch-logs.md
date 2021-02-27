---
layout: post
title:  "Docker Logs -> AWS CloudWatch"
author: VJ
categories: [ AWS, Docker ]
tags: [AWS, Docker, CloudWatch, logs]
image: assets/images/docker.png
description: "How to send docker logs to cloudwatch"
featured: true
hidden: false
comments: true
---

Let's say you host various microservices or applications with the help of docker containers. Since docker container logs are transient (i.e. you lose your data when you stop your container) you want to store it more durably. One way of doing it is to mount a file system from host and use file based logging into it. Since it is possible to run docker container in multiple hosts, there is still a possibility of scattered logs. Docker provides various drivers which can be used to log into systems like splunk etc. One such driver is `'awslogs'` that helps to push your container logs into AWS Cloudwatch.

  

Once you push your container logs into CloudWatch, you could leaverage CloudWatch metric filter, insights etc. to monitor and work with your log files. You also can use subscriptions to get access to a real-time feed of log events from CloudWatch logs and have it delivered to other services such as elastic search or lambda for further processing or analysis. You could also choose to export your data to Amazon S3 within the same account or to a different account.


### How to?

In order to use `'awslogs'` driver to push logs into CloudWatch, you can set docker's logging driver from JSON file logging driver to awslogs. You could make this change via `'/etc/docker/daemon.json'` or can override the default driver when running the container.

```JSON
/etc/docker/daemon.json

{
  "log-driver": "awslogs"
}
```
This should set the logging driver to  `'awslogs'`  for all the containers you run in the host.  You can do this just for selected containers by overriding the driver in the `'docker run'` command.

```
docker run --log-driver=awslogs ...
```

You can also choose the region in which you want to have your CloudWatch Logs. If you do not set this, by default docker will log it in the region where your container runs(when run in EC2).

To pass region 

```
docker run --log-driver=awslogs --log-opt awslogs-region=ap-south-1 ...
```
The same can be set in `'daemon.json'` file. 

Before running the docker container that push the logs to CloudWatch, you should create the log group in CloudWatch or should allow docker to create one by passing `'--log-opt awslogs-create-group=true'` in the `'docker run'` command.  For this docker should have privileges to create log group and also to stream logs. 

Below is a sample IAM policy with limited previleges.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:CreateLogGroup",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

If  docker is running in an EC2 instance, you can create an IAM role and attach the above policy.  Attach the instance profile to the EC2 instance that runs `'docker daemon'`. If you are not running `'docker daemon'` in an EC2 instance, you can provide credentials with the `'AWS_ACCESS_KEY_ID'`, `'AWS_SECRET_ACCESS_KEY'`, and `'AWS_SESSION_TOKEN'`  in `'~/.aws/credentials'`  file.

This should allow the docker to push logs into CloudWatch.

### Example:

```
docker pull vnair5/lottery-generator-service

docker images
```

![cicd]({{ site.baseurl }}/assets/images/docker_pull_screenshot.png)


Now run:

```
docker run -p --log-opt awslogs-group=ec2-docker  8090:80 12ef935a9efe
```

This should spin up the container and push logs into CloudWatch. You can check this by logging into AWS console and by opening CloudWatch:

![cicd]({{ site.baseurl }}/assets/images/docker_logs_cloudwatch_ss.png)

