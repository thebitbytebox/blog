---
layout: post
title:  " Gluco Diary - Angular 8, Spring boot Microservice & Docker demo project"
author: VJ
categories: [ Microservices ]
tags: [angular, microservices, REST, spring, springboot, java]
image: assets/images/gluco-diary.png
description: "Gluco Diary - Angular 8, Spring boot Microservice & Docker demo project"
featured: false
hidden: false
---
Gluco Diary is a small demo project for angular, spingboot microservices.

Project provides a simple ui for recording blood glucose readings. Application helps user to create and login to an account. Users can start recoding their blood glucose levels for both before and afterfood (Morning, Afternoon and Night).


*Note : The demo project is just a demo project and could improve a lot. I will keep updating the project whenever possible.*


## Front End

Front end for the application runs on Angular 8. The entire application is packaged inside a docker container. The compiled code will be running in a nginx webserver inside the docker.


| Desktop View | Mobile View |
| ------ | ------ |
| ![Screenshot_from_2019-07-31_22-24-36]({{ site.baseurl }}/assets/images/gluco-sc-1.png) | ![iphone]({{ site.baseurl }}/assets/images/gluco-iphone.png) |





## Back End

Backend services are in springboot with springboot netflix. There are two core services right now, one for user resitration\authnetication and another to record the blood sugar levels. 
There are additional services to support a microservice architecture. 

### Architecture

<img src="{{ site.baseurl }}/assets/images/gluco-archi.png" width="50%" style="display:block; margin:0 auto;">


**User-Service**


Used to register and authenticate a user to get a JWT token. Additionally, it has methods to validate a token, get user profile and logout. User micro service has its own database. Here we have used mongodb to store user details.
Once a user authenticates, a JSON web token is generated and stored against user profile. JSON web token is signed using a key and has an expiry which is configurable. All resources except login and register is secured and needs a token bearer authentication. 

*Note: I will slowly improve authentication process to make it more efficient.*


**blood-sugar-service**

This microservice is responsible for recording a users blood sugar level. This service has token based authentication enabled. It validates the token by calling user-service. 

## Infra Services

For infrastructure, we currently have spring cloud config server, eureka registry server and zuul proxy gateway.


**Cloud Config Server**

All services has their properties hosted in the cloud config server. The properties are currently hosted inside the service with native profile. [Sensitive datas are encrypted using a keystore.]
Cloud confg server has security enabled. Inorder to access the properties via browser, you need to authenticate yourself with a user name and password which you will configure.




**Zuul Proxy**

All communication from the UI is routed to services via the zuul proxy. Proxy has routes configured for user and blood-glucose services. blood-glucose-service uses  spring feign client to call user-service for token validation. This is done via look-up in the registry server.




**Registry Server**

All services register itself with the spring Eureka server. 


## Set Up

Follow below steps to run the application : 

### Pre-Requisties

*  Install JDK
*  Install maven
*  Install Docker 
*  Install Docker Compose


### Spring Config KeyStore

Project currently contains a java keystore under config server resources. This is used to encrypt the sensitive information when stored in the config files. 

So, Generate a java keystore and replace the file. Also, update the bootstrap.yml file with your filename.

`encrypt:
  key-store:
    location: classpath:/gluco-keystore.jks`


### Environment Variables

Many of the passwords and settings are configured via environment variable. Hence set up these in your machine environment.

* SPRING_PROFILES_ACTIVE=prod
* USERDB_PASS=<Add>
* USERDB_USER=<Add>
* GLUCODB_PASS=<Add>
* GLUCODB_USER=<Add>
* GL_KEY_PASSWORD=<Add [ This is your keystore password generated before ]
* CONFIG_SERVICE_PASSWORD=<Add> [ This is the password your spring config service will use to secure itself.]
* HZ_CAST_MGMGT_PASSWORD
* HZ_CAST_MGMGT_USER


### Run Maven

Run maven build to generate the FATJar files. We will pack these as a docker image soon.

`mvn clean build`

### Containarize

We will contianarize the jars that we created for each service. Every service has its own Dockerfile. This file is used to create the docker images. All these imaages will share a network. To make things simpler, we will use docker-compose
The root of the project contains a docker-compose.yml file with the docker instrutions. It has placeholders for various environemnt properties  which will be taken from your system environment settings.

Run

`docker-compose build` - This will build the docker images.

After creating the images, we will run the docker images by executying

`docker-compose up -d`

Note: -d will make the startup silent.



If you want to see the contianers, execute 

`docker ps`

To see logs for each container

`docker logs <containerid>`

## CI-CD

Currently the project has a config file for gitlab-ci. With every commit pipeline will be triggered to build and deploy the images. As a test I used digital ocean droplet to install gitlab-runner and automate the deployment of the services.

The CI happens in 3 different steps.


1.  Maven build is kicked off to buid and package services.
2.  Build docker images. For this we are using docker-compose file.
3.  Run the docker-compose to start the containers.

![cicd]({{ site.baseurl }}/assets/images/gluco-cicd.png)

Note: Angular build and package is done inside the Docker via Dockerfile. The final output files are copied into the nginx server.


## Next Steps


1.  Improvise user-service with authentication through cache.
2.  Add reporting-service to provide data for graphs and charts.
3.  Add notification-service to provide user notifications.
4.  Add distributed tracing.
5.  Add monitoring and reporting.