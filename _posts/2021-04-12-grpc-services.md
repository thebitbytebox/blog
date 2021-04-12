---
layout: post
title:  "gRPC based APIs"
author: VJ
categories: [ Microservices ]
tags: [gRPC, REST, Microservices, API, JAVA]
image: assets/images/grpc.png
description: "Why gRPC?"
featured: true
hidden: false
comments: true
---


gRPC developed by Google is gaining a lot of traction these days since it can use protocol buffers for data serialization. This makes the 
payloads smaller, faster and simple. Various tests shows gRPC to be much faster(more than 6x) than REST based APIs.

gRPC: 

- It is built on Protocol Buffer(Protobuf)
- Multiplex many requests with one connection through HTTP/2
- In built code generation for numerous languages
- Smaller payload
- Header compression
- Faster message transmission
- Binary protocol

A plain gRPC service could be slower in terms of implementation perspective since we are used to quick development of 
APIs with frameworks such as Springboot etc. However, with [grpc-spring-boot-starter](https://github.com/LogNet/grpc-spring-boot-starter)
I think we can quickly get around the problems in terms of slower implementation, service discovery, load balancing etc.

# REST vs gRPC

Both gRPC and REST based APIs has their own merits and de-merits. 


### REST 

- Simple and easy to implement.
- Can make requests directly on a browser.
- Uses human readable JSON messaging.
- Can use various documentation tools such as swagger to define and provide the consumers a contract. ð

However,

- Rest is based on HTTP 1.1. 
- No header compression.
- Bigger payload, thus higher bandwidth usage.
- Latency in processing requests.

### gRPC

- It is built on Protocol Buffer(Protobuf)
- Multiplex many requests with one connection through HTTP/2
- In built code generation for numerous languages
- Smaller payload
- Header compression
- Faster message transmission
- Binary protocol

However,

- Probably slower implementation
- Cannot call a gRPC service directly from a browser.
- Not human readable.


Now the question is when should we use a gRPC based APIs and REST based APIs.

I personally think, we could use gRPC based APIs for internal microservice. This makes the inter service communication faster and lighter. However, when we want to expose an API to a external third party consumer, REST API would be much better due to its human readability factor and better contract information. 

With gRPC we need to generate a stub (stub generation is provided out of box) in order to call the APIs. Hence if someone else wants to consumer our APIs, we need to either provide them with generated stub or provide them our protobuf files. Protobuf files would help the consumers understand about the contract and messaging format but might put an extra burden on them to generate stub from it. So, I say send them both.

Currently gRPC has support in the following languages: 

- C#
- C++
- Dart
- Go
- Java
- Kotlin
- Node
- Objective-C
- PHP
- Python
- Ruby

Using tools like [BloomRPC](https://github.com/uw-labs/bloomrpc) would help us with testing our gRPC servcies by generating a human readable request/response and avoid the stub generaion for testing purposes.

It supports APIs such as:

- Unary APIs (Regular request-response)
- Server side streaming
- client side streaming
- Bi-directional streaming

### gRPC in AWS

AWS recently started supporting HTTP2 with GRPC in their application load balancer. We could also use App Mesh in tandem with
Elastic Container Service to deploy, discover and implement a microservices mesh.

With gRPC, your client will use a stub that is generated from the proto to invoke the ALB with gRPC service as a backend. 
ALB will allow clients to talk to a mix of both gRPC and non-gRPC services target groups.

Read More on [AWS](https://aws.amazon.com/blogs/aws/new-application-load-balancer-support-for-end-to-end-http-2-and-grpc/)

Read more about gRPC on [grpc.io](https://grpc.io/)

## account-search

Project is a sample service developed with plain gRPC implementation. The service talks to a elastic search server to 
filter the data based on the input payload.

You can find the code on my [gitlab](https://gitlab.com/gunnervj/account-search-grpc)

To demonstrate server side streaming, we stream results from the elastic search 5 at a time. Hence, we are using search-after
functionality in the elastic search query.

Example: If there are 20 results based on the query, we query only for the first 5 from elastic search, then stream the 
results to client, then query again for the next 5 and stream it. This is repeated until we reach the last result.

This approach is good when you have thousands of results that needs to be sent to client. Streaming a partial set each 
time will make sure that we keep our resource utilization at bay.


The service is defined in the search.proto file.


### search.proto

```protobuf
syntax = "proto3";

package account;

import "account/account-search.proto";

option java_multiple_files = true;
option java_package = "com.bbb.grpc.account.service";

service AccountSearchService {
  rpc accountSearch(SearchRequest) returns (stream SearchResponse) {};
}


```

We are using gradle plugin to automatically generate the service definition code. 

Payload is defined in account-search.proto and account.proto files

### account.proto
```protobuf
syntax = "proto3";

package account;

option java_multiple_files = true;
option java_package = "com.bbb.grpc.account.beans";

message Account {
  string name = 1;
  string alter_ego = 2;
  string role = 3;
  string comic = 4;
  string affiliation = 5;
}
```

### account-search.proto

```protobuf

syntax = "proto3";

package account;

import "account/account.proto";

option java_multiple_files = true;
option java_package = "com.bbb.grpc.account.beans";

message SearchRequest {
  repeated SearchKey search_field = 1;
}

message SearchKey {
  string name = 1;
  string value = 2;
}


message SearchResponse {
  repeated Account accounts = 1;
  int32 records_found = 2;
}
```
``AccountServer.java`` is the entry point that needs to be run to start the gRPC server.

## How to Run


I have packaged everything as containers for the ease of running.

1) Create the java classes from the proto files
   
   ```
   ./gradlew generateProto 
   
   ```
   
   [ You can find the code under build/genrated/source/proto folder ]


2) Generate docker image

   ```shell
   docker build . -t docker build . -t <update your tag name> 
   ```

3) Create a docker network

   ```shell
   docker network create -d bridge account-bridge-nw
   ```

4) Export env variables

   ```shell
    export ELASTIC_HOSTS=http://elastic:9200
   ```
5) Run elastic search container

   ```
   docker run -itd -p 9200:9200 --network=account-bridge-nw --name=elastic  vnair5/elastic-search
   ```
    I have preloaded the container elastic search with index and data.
    
6) Run the gRPC service
    ```
    docker run -itd -p 50051:50051 --env ELASTIC_HOSTS --network=account-bridge-nw <update image:tag from step2> 
    ```

Now you can make requests to the service to test the same. 

## Ways to test:

- Generate stub from the proto files and make requests to the service

- Use tools like BloomRPC to test. [BloomRPC](https://github.com/uw-labs/bloomrpc)


## Screenshot

![result]({{ site.baseurl }}/assets/images/grpc-account-search/results.png)

Since the total number of records found is 12, there are 3 streams. 
The first 2 stream has 5 records each, and the final stream has 3 records.
