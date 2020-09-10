---
layout: post
title:  "Hazelcast IMDG"
author: VJ
categories: [ Microservices ]
tags: [angular, microservices, REST, spring, springboot, java, cache, hazelcast, imdg]
image: assets/images/hazelcast.png
description: "Hazelcast IMDGhazel"
featured: false
hidden: false
---

In our demo project([GlucoDiary]({% post_url 2019-07-27-Gluco-Diary%})) we are using a JWT token to authenticate the user for post login API accesses. However, we are storing these JWT tokens against user in the database. Validating the token for each requests needs a database call which is very expensive, specially when we have millions of users.

Well, we could just limit our token validation to its signature and expiry.But that wont be fully secure as our token is vaid for one hour and someone could misuse it, if they get hold of it.  To avoid this, post login we store our token into our database gainst the user. Upon user signout, we remove the token from the database which will lead to “Authentication Error”, if at all someone try to use it. (We could use access and refresh tokens to limit life of access token. But lets work with the current situation in our small app.)

## How can we make this better?

Well,

- What if we could store this in memory which has a faster read/write?
- What if we can scale this depending on our need?
- What if it is fault tolerant?
Yes, we can use an In-Memory Data Grid or IMDG for our problem. There are numerous IMDG platforms available in market. I have picked Hazelcast for our project as it is fast, simple and efficient.

### Hazelcast Features
- **In-Memory Data Grid** : Hazelcast IMDG is often used as an operation memory layer for databases in order to improve application performance  and to distribute data across servers. Hazelcast can ingest data at very high rates, and also can manage large data sets.
- **Operationl Memory** Hazelcast IMDG can be used as the operational memory of a Microservices architecture.
- **In-Memory No SQL** : Can be used an In-Memory NoSQL solution
- **Messaging Platform**: A simple messaging platform with a publisher subscriber model.
- **Cache**: Used as a regular application cache with replication.
- **Web Session Clustering**: to maintain user sessions in-memory for redundancy and seamless backup.


We can implement hazelcast as a standalone application to form a cluster or as an embedded implementation. Both has its own advantages and disadvantages. For example, if you want to update your application. With an embedded solution, you will have to make sure you have atleast 2 nodes in cluster and you update one by one. Bringing appliation down will discard all values in hazelcast. But this kind of implemention has some data on-premise which will be faster when compared to the standalone implementation. However, standalone impletmentation has better availability and maintenance. But could have an impact on performance with network latency etc.

We, in our application will go with the embedded implementation.

To learn more about hazelcast and its amazing features, visit Hazelcast Training

I will post an update to the git with the hazelcast implementation soon with v1.1. Till then.

Adios.

