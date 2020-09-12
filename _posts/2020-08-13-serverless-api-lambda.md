---
layout: post
title:  "Serverless API"
author: VJ
categories: [ Serverless ]
tags: [REST, API, serverless, AWS, lambda, api gateway, dynamodb]
image: assets/images/lambda-api.png
description: "A Serverless API using node and dynamodb"
featured: true
hidden: true
comments: true
---

First let's see what we are going to develop.

We are going to develop an API that will helps us to track a baby's weight. This API will support adding a new baby to track his/her weight. It also supports deleting, editing babies. Mostly what we are dealing here is the CRUD operations. To store the baby data, we will use AWS dynamodb as the backend data store.

We will develop these functions in nodejs and then use serverless framework to provision in AWS.

start with ```npm init``` and go through the series of questions to create a node project

Install following dependencies

- moment
- uuid

example :  ```npm i moment --save```

This is going to save momentjs as one of our dependency.

We also need aws-sdk, however we will keep this as a dev dependency. This is because, to run a fucntion in lambda, AWS has the library and we do not need to provide it in our bundle.  You can do this by using --save-dev when issuing the command. example: 

```npm i aws-sdk --save-dev```

This will also reduce our bundle size and will help in faster execution of the function. When your code package size is less, aws lambda will load it faster as it provides less load to the contianer to spin up when it is coming out of idle state.

Now create a folder ```src``` to hold our js files.

We will also create a serverless.yml file at the root of our project. This yml file will contain the serverless shorthands to create resources we need.

Before getting into the node js development, lets fill in some details to the serverless.yml

Add below lines into it

```yml
service: baby-weight-tracker-api

frameworkVersion: '>=1.0.0 <2.0.0'  

provider:
  name: aws
  runtime: nodejs12.x
  stage: ${opt:stage, 'dev'}
  region: ${opt:region, 'ap-south-1'}
```

Here, we are declaring some common configs by saying that our runtime is nodejs and our provider is aws and is in ap-south region. We can pass stage, region at runtime to configure different stages or regions. If nothing is provided, it will take the default value.


To store our baby data, we will use dynamodb and we will use serverless to provision it. We declare the resources that we want to provision under the Resources tag. I like to export some of these declarations in a separate file to make it organized. For this create a new folder under the root directory and name it as resources. We will hold our additional serverless yml here under this folder.

For dynamodb, create a new file say, ```dynamodb-resource.yml``` and add the below lines

```yml
Resources:
    BBBUserBabyTable:
        Type: AWS::DynamoDB::Table
        Properties:
          TableName: ${self:provider.environment.USER_BABY_TABLE_NAME}
          AttributeDefinitions:
            - AttributeName: user_id
              AttributeType: S
            - AttributeName: baby_id
              AttributeType: S
          KeySchema:
            - AttributeName: user_id
              KeyType: HASH
            - AttributeName: baby_id
              KeyType: RANGE
          ProvisionedThroughput:
            ReadCapacityUnits: 1
            WriteCapacityUnits: 1
```

Here we are provisioning a new DynamoDB table with a composite key of ```user_id ``` as the ```HASH``` and baby_id as the ```RANGE``` key. The reason why I chose these fields is because dynamodb works well when we query it using the partition key, which is the ```HASH```. Most of our queries to database would be based on the babies of a particular user, hence we have made ```user_id``` as the ```HASH```. But we also need to select a particular baby, hence we used ```baby_id``` as the ```RANGE``` key. It essentially does not have to scan the whole table when searching for baby for a particular user. It just goes to a particular partition with the user hash and retireved the baby or babies under the user.

I have only mentioned the key fields that I require in the table. Rest of the data can be added at runtime and that is the beauty of it. We do not have to specify all the columns when creating a table like in the traditional RDBMS. 
 
The table name is coming from an environment variable. Declare that by adding the below lines under the provider. This will append the stage name into the user-baby- table name. This would help us having different tables for different stages like user-baby-dev , user-baby-prod etc.

```yml
provider:
  name: aws
  runtime: nodejs12.x
  stage: ${opt:stage, 'dev'}
  region: ${opt:region, 'ap-south-1'}
  environment:
    DYNAMO_REGION: ${opt:region, 'ap-south-1'}
    USER_BABY_TABLE_NAME: user-baby-${opt:stage, self:provider.stage}
```

Now, include the ```dynamodb-resource.yml``` into our main serverless.yml file. This should be in the same level as ```provide```, ```service``` etc. YML is particular about the intendation, so be cautious.

```yml
resources:
  - ${file(resources/dynamodb-resource.yml)}
```

Now we are going to develop the functions to perform our CRUD operations. Lets' start with the function to create the baby in our database. 

Create a file ```add-baby.js``` just for adding a baby. Separating functions to its own file will help us to package each functions separate thereby keeping a small package size.

add the imports 

```javascript
const uuid = require('uuid');
const util = require('./util');
const moment = require('moment');
const dynamoDBServerless = require('serverless-dynamodb-client');
```

Remember we passed our table name as an environment variable? We can get that  by

```const userBabyTableName = process.env.USER_BABY_TABLE_NAME;```

Here, if you see, we are using serverless-dynamodb-client instead of AWS DynamoDB DocumentClient. This will help us later in using offline dynamodb for our development. So we need to add this to our dependencies.

```npm install serverless-dynamodb-client --save```

Our request is going to look like this

```json
{
    "baby_name": "Tutu",
    "gender":"F",
    "dob":"07-06-2020"
}
```

Lambda handler acn be defined as ```exports.handler = async (event) => {```

We will get our request details and other information in the event object. We are going to do some request validation and then create an item for the baby which we will store into dynamodb. I have moved the validations and some other common functions into a separate file ```util.js```.

### util.js

```javascript
const moment = require('moment');

const getResponseHeaders = () => {
    return {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': '*',
        "Access-Control-Allow-Credentials": true
    };
};

const getUserId = (event) => {
    return event.requestContext.identity.cognitoIdentityId;
};

const constructError = (errorCode, httpCode, errorMessage) => {
    let error = {
        statusCode: httpCode,
        headers: getResponseHeaders(),
        body: JSON.stringify({
            errorCode: errorCode,
            errorMessage: errorMessage
        })
    };

    return error;
};


const contructBabyResponseBody = (baby) => {
    let body = {
        "baby_id": baby.baby_id,
    }
    if (baby.baby_name) {
        body.baby_name = baby.baby_name;
    }
        
    if (baby.gender) {
        body.gender = baby.gender;
    }
    
    if (baby.dob) {
        body.dob = baby.dob;
    }
    return body
};

const isDOBInvalid = (dob) => {
    return dob === undefined ||
            dob === '' ||
            !moment(dob, 'DD-MM-YYYY').isValid() ||
            moment(dob, 'DD-MM-YYYY').isBefore(moment().subtract(1, "years"));
};

const isDOBInvalidFormat = (dob) => {
    if (dob !== undefined && dob !== '' ) {
        return !moment(dob, 'DD-MM-YYYY').isValid() ||  moment(dob, 'DD-MM-YYYY').isBefore(moment().subtract(1, "years"));
    }
};

const isBabyWeightInValid = (weight) => {
    return weight == null || weight == undefined
                || !Number.isFinite(weight) ||
                weight < 0 || weight > 20;
};

const isRecordDateInvalid = (date, dob) => {
    if (date === undefined ||
        date === '' ||
        !moment(date, 'DD-MM-YYYY').isValid()) {
        return true;
    } else if (!moment(date, 'DD-MM-YYYY').isBetween(moment(dob, 'DD-MM-YYYY'), moment(),undefined, '[]')) {
        return true;
    }
};

const isBabyNameInValid = (baby_name) => {
    return baby_name === undefined || baby_name.trim().length === 0;
};

const isGenderInValid = (gender) => {
    console.log(gender)
    return gender === undefined ||
            gender.trim().length === 0 ||
            (gender.toUpperCase() !== 'M' && gender.toUpperCase() !== 'F');
};

const isGenderInvalidFormat = (gender) => {
    if (gender !== undefined && gender !== '' ) {
        return gender.toUpperCase() !== 'M' && gender.toUpperCase() !== 'F';
    }
};


module.exports = {
    getResponseHeaders,
    getUserId,
    constructError,
    isDOBInvalid,
    isDOBInvalidFormat,
    isBabyNameInValid,
    isGenderInValid,
    isGenderInvalidFormat,
    contructBabyResponseBody,
    isRecordDateInvalid,
    isBabyWeightInValid
};
```


### add-baby.js

```javascript
exports.handler = async (event) => {
    try {
        let baby = JSON.parse(event.body);
        if (util.isDOBInvalid(baby.dob)) {
            return util.constructError(101, 400, 'Invalid Date of Birth(Baby should not be older than a year)');
        } else if (util.isBabyNameInValid(baby.baby_name)) {
            return util.constructError(102, 400, 'Invalid Baby Name)');
        } else if (util.isGenderInvalidFormat(baby.gender)) {
            return util.constructError(103, 400, 'Invalid Gender');
        }

        const item = {
            user_id: util.getUserId(event),
            baby_id: uuid.v4(),
            baby_name: baby.baby_name,
            gender: baby.gender,
            dob: baby.dob,
            created_at: moment().toISOString(),
            weight_chart: {}
        };
        await dynamoDB.put({
            TableName: userBabyTableName,
            Item: item
        }).promise();
        headers = util.getResponseHeaders();
        headers.baby_id = item.baby_id;
        return {
            statusCode: 201,
            headers: headers,
            body: JSON.stringify(util.contructBabyResponseBody(item))
        };

    } catch (error) {
        console.log('Error Occured ', error);
        return util.constructError(104, 500, 'Something went wrong. Try later. If problem persist contact support.');
    }
};
```

If everything goes well, then we are going to send a response with Http Status Code 201 and will also set the created baby_id in the request header. Also, return the baby information back to the caller in the body.

Normally we to test this, we need to run the code in aws by creating a lambda function. However, we could run this in our local by using serverless offline plugin. With the serverless offline plugin you can speed up local dev by emulating AWS lambda and API Gateway locally when developing your Serverless project.

First install the plugin 

```
npm install serverless-offline --save-dev
```

Add the plugin to the serverless.yml

```yml
plugins:
  - serverless-offline
```

You can now run sls offline to start. However, we still need to do some more configuration in our serverless.yml file.

- Define lambda functions
- Define Roles for lambdas to access dynamodb

Let's first start with the roles.

Our lambda needs what some permissions to perform various operations in the dynamodb table?  Well, we would require to 

1. PutItem to add a new baby
1. GetItem for retrieving the baby details
1. DeleteItem for deleting a baby
1. UpdateItem to update detials
1. Also Scan and Query to query and scan the tables.

FOr this under the provider, add below lines

```yml
iamRoleStatements:
    - Effect: 'Allow'
    Action:
        - dynamodb:DeleteItem
        - dynamodb:PutItem
        - dynamodb:GetItem
        - dynamodb:Scan
        - dynamodb:Query
        - dynamodb:UpdateItem
    Resource: 
        - "Fn::GetAtt": [ BBBUserBabyTable, Arn ]
```

```"Fn::GetAtt": [ BBBUserBabyTable, Arn ]``` this will assign the ARN of the dynamodb table created by serverless and attach it to the roles allow statement.

We declare this under the provider so that this role is added automaticaly to all the functions we define. However, if you prefer to give separate refined permissions for each functions, we could do so too.

Now lets start defining the lambda functions and the API gateway paths. 

### Lambda Function declaration with API gateway resource

With the same level as ```provider```, add below line

```yml
functions:
  add-baby:
    handler: src/add-baby.handler
    description: Function to add a baby into database
    events:
      - http:
          path: babies
          method: post
          authorizer: aws_iam
          cors: true
```

This will add a new lambda function add-baby and point it to the handler in ```add-baby.js``` . Also attach the function to the API Gateway with resource path as ```/babies```. We have added authorizer to be aws_iam which will add IAM authentication to access the resource. Hence the consumer of the API should be able to authenticate and acquire a token with previleges to access an AWS resource. 

We will use AWS congnito user-pool to register the users for the application. ANd we also require an identity-pool which will be using the userpool for the token access in order to access aws resource. I am skipping this setup for now and will write as a separate post.

You can now run the command ```sls deploy``` to deploy the resources to aws or run ```sls offline```.

Note: We need to have dynamodb table in aws in order for this to work. Else you will see errors.

### Dynamodb offline

We now will be able to test the lambda function via api gateway either from aws or via sls ofline plugin in local. However, in both the cases we need to have the dynamodb table created in the aws cloud env. So lets add dynamodb offline plugin to this in order to have a local serverless dynamodb.


Run : ```npm install --save-dev serverless-dynamodb-local```

Add ```serverless-dynamodb-local```  under the plugins in ```serverless.yml```

```yml
plugins:
  - serverless-dynamodb-local
  - serverless-offline
```

Make sure ```serverless-offline``` is added at the end in the list.

***To use the plugin***

- Install DynamoDB Local by runinng ```sls dynamodb install```
- Start dynamodb local by running ```sls dynamodb start``` . This will essentially start local dynamodb on port ```8000```

We have an admin console which comes very handy when coming to dynamodb.

***Setup***

```
npm install dynamodb-admin -g
export DYNAMO_ENDPOINT=http://localhost:8000
```

Run ```dynamodb-admin``` to start the admin client

That's it. You can run the project by executing ```sls deploy```

If you open cloudformation console, you should see your stack being created.

***Cloudformation***

![Screenshot_from_2019-07-31_22-24-36]({{ site.baseurl }}/assets/images/cloudformation-baby.png)

***dynamo-db***

![Screenshot_from_2019-07-31_22-24-36]({{ site.baseurl }}/assets/images/dynamodb-baby.png)

***API Gateway*** <br>

![Screenshot_from_2019-07-31_22-24-36]({{ site.baseurl }}/assets/images/api_gateway_baby.png)


You can find the project in [My Git](https://gitlab.com/gunnervj/baby-weight-tracker-api).

