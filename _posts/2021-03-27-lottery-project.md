---
layout: post
title:  "Serverless Micro Services With AWS Fargate and Spring Boot"
author: VJ
categories: [ AWS ]
tags: [AWS, CloudWatch, logs, Cloudformation, Infrastructure As Code, yaml, yml, vpc, subnet, routes, application load balancer, ALB, Docker, Xray, Fargate, ECS, service discovery, route 53]
image: assets/images/docker-fargate-boot.png
description: "Serverless Micro Services With AWS Fargate and Spring Boot"
featured: true
hidden: false
comments: true
---

In this modern world using serverless applications helps us not worry about provisioning servers, 
maintaining them, making them highly available, scaling according to load etc. We only need to worry about hardening 
and fixing security holes in our application but not servers running them (We still need to secure our other 
infrastructure).

Spring boot packaged in a docker container forms a powerful combination to develop and run microservices with agility.

AWS Fargate helps us run our docker containers in a serverless fashion without provisioning EC2 servers to run docker
or ECS clusters.

This Project is created to demonstrate running spring boot micro-services in AWS Fargate - Totally Serverless

It uses following technologies:

- Spring Boot Microservices
- S3
- AWS Java SDK
- Cloudformation for Infrastructure as Code
- Dynamo DB
- AWS ECS Service Discovery
- Route 53
- Cloud Watch Logs
- AWS Xray
- AWS Application Load Balancer with path based routing


Find the code in [Gitlab](https://gitlab.com/gunnervj/lottery-project)

## Architecture

![Architecture]({{ site.baseurl }}/assets/images/docker-fargate-boot/architecture.png)


## Services


Project consist of 3 spring boot micro services

- lottery-service
- printer-service
- ticket-service

These services runs on spring boot tomcat server on a docker container with exposed port 80.

#### Lottery Service

This micro service is exposed to the users via application load balancer. Users can use below API to buy a lottery ticket.

``
HTTP-POST  http://<alb-host>/bbb-lotto/api/v1/lottery/buy
``


This micro service will in-turn calls ticket-service to generate a lottery ticket number. It then calls printer service to print the lottery with a QR code and a ticket number. It then saves some meta information to the dynamodb table. The API sends a response with the s3 signed url received from the printer service. This URL can be used to download the ticket.

This URL is only valid for 15 minutes and cannot be retrieved later.

#### Ticket Service

This micro service is responsible for generating the lottery ticket number via API

``
HTTP GET http://<internal-service-discovery-host>/ticketing/api/v1/ticket
``

#### Printer Service


This micro service takes in a lottery ticket number and lottery ticket id as input parameter via exposed API

``
HTTP POST http://<internal-service-discovery-host>/printer/api/v1/print
``

The service then uses a ticket blueprint from S3 and imprints a QR code generate from lottery-id in the ticket blueprint. It then prints the ticket number in the blueprint and finally upload the image to a S3 bucket. Before responsding back, the service generates a s3 signed url that is valid for 15 minutes and sends the url in the response.

![ticket]({{ site.baseurl }}/assets/images/docker-fargate-boot/lottery-ticket.png)

## Service Monitoring

AWS XRay daemon runs as a side car container with each service. Services are instrumented with XRay SDK in order to generate traces which will be uploaded via XRay daemon.

Each container also sends application logs to AWS CloudWatch logs via ``awslogs`` driver.

####  XRAY

![XrayMap]({{ site.baseurl }}/assets/images/docker-fargate-boot/xray.png)
<br/><br/><br/>
![XrayServiceInfo]({{ site.baseurl }}/assets/images/docker-fargate-boot/xray-svc.png)

#### CloudWatch

![CW-map]({{ site.baseurl }}/assets/images/docker-fargate-boot/cloudwatch-servicemap.png)
<br/><br/><br/>
![CW-logs]({{ site.baseurl }}/assets/images/docker-fargate-boot/cloudwatch-logs.png)

## Service Discovery

Lottery-Service registers itself to a target group to which traffic is routed from ALB via path based routing. Ticket and Printer services are called from the lottery-service through internal service discovery based hostnames in Route 53. For service discovery we are using WEIGHTED policy via A record since we are exposing our services via port 80.

Read more on [AWS Page](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)

#### Route 53

![Route-53]({{ site.baseurl }}/assets/images/docker-fargate-boot/route53.png)

## Cloudformation - Infrastructure As Code

We use cloudformation to provision our resources in this demo-project

Under cloudformation project you can find two folders named

- Infrastructure
- services


Infrastructure folder contains cloudformation templates to provision VPC, application load balancer, ecs-cluster, roles and security groups necessary for the services to run.

These are run together using the ```infra_master.yml``` in the cloudformation folder.

Similarly, under services we have cloudformation templates for each of our services which creates task definitions, ecs services, service discovery etc.

These can be run via the ```services_master.yml``` cloudformation template which creates the entire stack.

I have created separate template for each service for simplicity of demo project. We could use one single service template that can be configured via parameters. Also, we could levrage AWS Code build and Code pipeline to build and deploy ECS services.


![CloudFormation Stack]({{ site.baseurl }}/assets/images/docker-fargate-boot/cloudformation-stacks.png)


## How to Run

1. S3 Preparation
    - Create 1 S3 bucket with any name.
    - Create a folder under the bucket with name as ``template``.
    - Upload the ``ticket_template.png`` under the printer-service to the folder ``template`` folder in the bucket.

    ![CW-logs]({{ site.baseurl }}/assets/images/docker-fargate-boot/s3-bucket.png)

2. Setting up core infrastructure
    - Upload the cloudformation templates into another S3 bucket.
    - Update the Template URLs based on the bucket you created in ```infra_master.yml``` and ```services_master.yml```. We could parameterize this in future.
    - Run the ```infra_master.yml``` cloudformation template to create the core infrastructure stack. Once this is completed, we have provisioned all the infrastructure that is needed to run our services.

3. Run Services by running ``services.yml``
    - Pass the s3 bucket name in the parameter ``S3BucketName``. If not it will use the default value of bbbox-lottery-bucket which will fail when the service tries to download the template from the s3 bucket while loading.

Once the services stack is created, make a post request to the below URL

``HTTP POST http://lottery-project-alb-1877314996.ap-south-1.elb.amazonaws.com/bbb-lotto/api/v1/lottery/buy``


This should respond back a json with url to download the image

```JSON 
{
    "location": "https://bbbox-lottery-bucket.s3.ap-south-1.amazonaws.com/b76123d4-448d-41a2-9307-fb9163d5eaa6.png?X-Amz-Security-Token=IQoJb3JpZ2luX2VjEOP%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCmFwLXNvdXRoLTEiRjBEAiA0xmoC7V73VZgcJOQ6Htz9iX0ly%2FrGkvPsZQQk5kWLYgIgDZcZKpUnBFtlGZCrWn4kI3EZU5GfwS69DxGIIHvkkooqyAMILBABGgwzODE5NzczOTIwMzciDMyvyErydOv0SsJwbSqlA9RPu6tsL8G%2Bd2sz9eh2jDCa8f7VBqhCx25BnwSghTTz80JE8XjGaGR1kKaKG9C5UB6g6hDYPU%2BYM9KVIo0UU%2B66dRJ%2FWQ9Df8haAuJ2HNiaIoAoOjluopZsRQZ%2BHDylT2A9KfY9UcD1EF%2FpYxzDbFOOM380bA048qPW6e%2FnVPXbQOvFRhn78SB4UjwVeHuV66LBFiFfdOTbzHDF5lkfJ22ihufVfKbmgFLHcgIHlsKI8gl9hgXzfLA4hxZOK2yIIOg%2FzT6teCPCPR5lAeohkqUYWVUu9XFJDwB09EzltGdQ3V%2FhSNK8zTCH2JAp%2FXeZsxnwVXQiYBx0llCt9GVOOitZ%2BTLNDrpwFg6Xr7J7JJmx0C%2BTz4jk8dWNL9nl85zOEsa18fskfLutzuALaVtLYH45mfG1Mx%2Fb6mchrDi41KQ4hQmFUcYe8%2BvV7mf4n9KhfEDmRvzZ%2F5zBozxKJMLFBSVgqRLAsK3dZwDoNi8tZrE8v0DM%2BemkIjs%2F06EyXykxi89w%2Fmcb4ncc9qFdjEouQbWhAGRMZtgi1lAPPhYX8qWgHOohzVgw5J78ggY67QHjJ5%2FJhna8DfTTCpYKzKEYMsi1hurP8vrv2aPzrSoub62U4Sq4SyLI1PxKeblYfYyYQU4d6egj6jfpaV26eBOtYk3vCcftNpnew%2BfVHKUJQx66AIkGBFDh%2FmzzqOF%2BZruEVCrHc4TJTV1QW6oWoXdICvaAmyyVm6jweDoypuKRVPdCQUbkSTap1x7YYaw1wMfGGOg03XZTNRtXtU089JSnTUADu8Q3nCVm0t8H4hkjppslpWz%2BGOdxMp6oFbLPlUnn30kcnLipI2KYwGJ65xMUvC2tU4sx2VD0mlJaui5RmJn963o6b30tSpR91e0%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20210327T111837Z&X-Amz-SignedHeaders=host&X-Amz-Expires=900&X-Amz-Credential=ASIAVR35AOOS4O4ZVOV3%2F20210327%2Fap-south-1%2Fs3%2Faws4_request&X-Amz-Signature=2e91ebf072482aab10347aa7d4e4ab09dacbf1b3262a4fbe6b68661f88c2fdb8"
}
```

![postman]({{ site.baseurl }}/assets/images/docker-fargate-boot/postman.png)

## Screenshots

#### ECS Cluster & Tasks

![Cluster]({{ site.baseurl }}/assets/images/docker-fargate-boot/cluster-tasks.png)


## Improvements

1. Enable CICD
2. S3 bucket creation automation for template and to store generated images
3. one template for services by parameterizing hardcoded values
4. Use AppMesh for discovery services and more.
5. UI to generate a lottery request.
6. Move ECS cluster to a private subnet.
7. Add autoscaling based on metrics.
