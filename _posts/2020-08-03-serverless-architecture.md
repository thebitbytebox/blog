---
layout: post
title:  "Serverless architecture"
author: VJ
categories: [ Serverless ]
tags: [REST, API, serverless, AWS, lambda, api gateway]
image: assets/images/serverless-logo.png
description: "A quick introduction to the serverless framework"
featured: true
hidden: true
comments: true
---

We all have delt with the menance of provisioning servers and taking care of scaling aspects. But what if we could run our code without provisioning any servers? Wouln't that be simple ?

AWS came up with lambda functions where you could run you code in cloud without the tasks of provisioning servers and configuring scaling etc. With offloading such tasks to AWS, we could concentrate more on our application rather than on infrastructure. Also it helps with to easily come up with the with iterations for our production releases in a quick turn around of time. 

There are many product in AWS stack that supports the serverless approach. Serverless products fall into all categories such as compute, storage, databases, analytics etc. AWS provides SAM which helps us in building serverless applications without dealing with manual provisioning. Something we call as infrastructure as a service. 

SAM - Serverless Application Model, is open source framework and provides shorthand syntaxes to build serverless applications. You can read about SAM by clicking [here](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-reference.html#serverless-sam-cli).

Now many cloud providers have their own serverless products. However, I would say AWS still leads the industry with the headstart they have.

### Serverless Framework

You can develop, deploy, troubleshoot and secure your serverless applications with radically less overhead and cost, that too a framework that is not tied to a provider. Serverless Framework helps you with that. What I love about serverless framework is not just that they provide a framework not tied to one provider but the documentation and user community they have. You can visit [serverless.com](https://www.serverless.com/) to get started.


Once you install the CLI for serverless,

setup the AWS config credentials

```
serverless config credentials --provider provider --key key --secret secret
```
Serverless uses this credentials with to provision lambdas and other resources you define. The user should have an admin role. Make sure you create a user for serverless and please, do not use the root user.

Once AWS credentials are configured, you can get started with the projects.

```
sls create --template aws-nodejs --path myService
```

You could use serverless or sls for shorthand to run serverless commands. In this example we are creating a nodejs project using an already available template. ```--path``` or ```-p``` is the path to which the project has to be created.

To deploy the application, you can simply issue the command ``` sls deploy ```

To remove a deployed application issue ``` sls remove ``` from the project directory.


I normally create my project first and then manually create the serverless.yml file to define my services and configuration.

We will quickly create a serverless nodejs project for an API. We will use serverless framework to define and deploy this to AWS.

Let's regroup in my next post.
