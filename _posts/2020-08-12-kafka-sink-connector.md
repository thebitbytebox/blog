---
layout: post
title:  "Kafka Connect Sink Connector"
author: VJ
categories: [ Kafka ]
tags: [kafka, elasticsearch, connector, confluent, java, sink, elk]
image: assets/images/kafka.png
description: "A Demo Project for Kafka Connect Sink Connector."
featured: true
hidden: true
comments: true
---
As wisely said, why re-invent wheels? Kafka connect helps us with something like that.

When using Apache Kafka, the source of the data could be a kafka-producer who pushes data into the kafka. The data could be from a database, application, cache etc. 

Let's take an example of loading data to a kafka topic from a database. This could be a broiler plate code. Many of us might have already wrote something like this. Why not re-use?

Kafka Connect help us with this scenario. They are ready to use components which can help us import or export data between kafka and external systems.

- A source connector help us with loading data into kafka from various sources
- A Sink connector help us to ship data out of kafka, say into elastic search or any other data sinks.


We already have many connectors which fit most use cases. Some are opensource, which others are supported by Confluent or its partners. 

Check out : [Confluent](https://docs.confluent.io/current/connect/index.html)

For available connectors: [Connectors](https://www.confluent.io/hub/?_ga=2.61129070.312483398.1596463018-44691259.1595303352)

But, when the avialable plugins does not fit our requirement, we sometime might need to write our own connector. This project is to demonstrate such a requirement. We transform source data from one format into another, perform some validations and then load into elastic search.

The project has lots of room for improvement and is for demo purpose for a standalone environment.

**Pre-requistes**

- Have docker and docker-compose installed in the running machine. As we are going to use containers.

**Set Up**

- Checkout the project.
- For linux/mac users, run the demo.sh
- Data to be published into kafka can be found under test-data.txt


Maven Arch Type : [https://github.com/jcustenborder/kafka-connect-archtype](https://github.com/jcustenborder/kafka-connect-archtype)


**URLS**

1. **Elastic search** : Can be access via port 2181. Docker port has been mapped to host 2181 port
2. **Kibana** : Has been mapped to host port 5601. URL :  http://localhost:5601/app/kibana
    - Sample Query: 
        GET /accounts/_doc/ACT002
        {
            "query": {
            "match_all": {}
            }
        }
3. **Kafka** : Has been mapped with 2 listeners, Internal and external. For AdvertisedListeners, Internal has been mapped with hotname as kafka and port as 19092. This is for applications within the docker network elastic-kafka. For external producers and consumers, advertised listener is set to localhost with port 9092. You can use this to publish messages into kafka topic.
4. **Kafka Connector** : The image is built using Dockerfile. This uses internal advertised listener as it is within the docker network. 


**Dockerfile**

Is built from image cp-kafka-connect-base. Base image does not contain any readymade connectors. Out built files are copied into etc and share folders. 'share' folder has our required libraries to run our custom connector. Hence this folder is mapped under plugins.path in worker.properties. However, we have set CLASSPATH to this folder in the CLASSPATH env variable for some of our libraries to load.

Entry point to this docker image is connect-standalone.sh from kafka with parameters, worker.properties and MySinkConnector.properties


**demo.sh**

- Running this will do a maven clean build.
- exports all mount paths for elastic search, kafka etc
- delete any existing docker containers and image for our demo project.
- build a new docker image
- Run a docker compose command to bring up all the services. If you want you can run this in detached mode by **docker-compose up -d**

Find the code in [Gitlab](https://gitlab.com/gunnervj/demo-sink-elastic)