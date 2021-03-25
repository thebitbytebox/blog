---
layout: post
title:  "AWS Amplify - Manual configuration"
author: VJ
categories: [ AWS, Amplify, Angular ]
tags: [REST, API, serverless, AWS, lambda, API gateway, dynamodb]
image: assets/images/amplify.png
description: "Custom configuration of Amplify in Angular"
featured: false
hidden: true
comments: true
---

AWS amplify is a set of tools that enables frontend and mobile developers to build fast, secure, scalable full stack application. You could easily build up authentication, data store, api calls etc from you application powered by AWS.
 
Amplify supports wide range of frameworks from both web and mobile.

**Web**

- Javascript
- React
- Angular
- Vue
- Next.js

**Mobile**

- Android
- iOS
- React Native
- ionic
- Flutter


It also provides you with some out of the box UI components for Authentication, storage etc for a wide range of frameworks.


![amplify]({{ site.baseurl }}/assets/images/amplify_develop.png)


### Amplify CLI

With Amplify CLI you could easily provision a new cloud backend with features such as authentication, APIs (REST and GraphQL), Storage, Functions and Hosting.

You can initialize a new Amplify project by running ```amplify init``` which takes you through a series of steps in configuring your Amplify project. 

However, we could also configure the Amplify library manually by adding some configurations.

Below steps shows us how to add Amplify into an Angular front end app. We will configure Auth and API package to call REST endpoint in AWS API gateway. We won't be using any Angular specific components, but only javascript functions.


#### Install amplify

Install amplify by running ```npm install -g @aws-amplify/cli```

#### Configure Amplify CLI

You can go through [Configure Amplify CLI](https://www.youtube.com/watch?v=fWbM5DLh25U)


#### Adding Amplify into an Angular project manually

- Install amplify libraries ```npm install --save aws-amplify```

***Specific for Angular***

Include ```node``` compiler option under ```src/tsconfig.app.json```

```json
"compilerOptions": {
    "types" : ["node"]
}
```

In ```pollyfills.ts``` add below line on top of the file

```javascript
declare global {
    interface Window { global: any; }
}
window.global = window;
````



- In ```main.ts``` file we will now configure Amplify

For this  we will import 

``` javascript
import { Amplify } from 'aws-amplify';
import { API } from 'aws-amplify';
```

> Note: Even though we are calling Amplify.configure() and passing configuration for API and Auth, we need to call API.configure() if not you will get error saying amplify is not configured when trying to call the API. Configuring Auth separately will result in unsigned API calls and will lead to Unauthorized errors.

For properties, we could user environment files.

```javascript
Amplify.configure({
  Auth: {
    region: environment.aws_region,
    userPoolId: environment.aws_userpool_id,
    identityPoolId: environment.aws_identity_pool_id,
    userPoolWebClientId: environment.aws_app_client_id
  },
  API: {
    endpoints: [
      {
        name: environment.api_name,
        endpoint: environment.api_endpoint_domain,
        region: environment.aws_region
      }
    ]
  }
});

API.configure({
  API: {
    endpoints: [
      {
        mandatorySignIn: true,
        name: environment.api_name,
        endpoint: environment.api_endpoint_domain,
        region: environment.aws_region
      }
    ]
  }
});

```

This will help us to use different settings for different environments.

> The code above imports the entire Amplify library. You can use separate imports like import Auth from '@aws-amplify/auth' to reduce the final bundle size.


***environment.ts***

```javascript
export const environment = {
  production: true,
  'aws_region': 'ap-south-1',
  'aws_userpool_id':  'ap-south-2_456456456',
  'aws_app_client_id': '180tgm3sdfsdfsd5huytrtydnfl7ig',
  'aws_identity_pool_id': 'ap-south-1:bef1e4b5-a3e6-443c-a4f0-54fdgdg4545',
  'api_endpoint_domain': 'https://api.thebitbytebox.com',
  'api_name': 'baby-weight-tracker-api-dev',
  'api_version': '/v1',
  'get_babies_path': '/babies',
  'record_weight_path': '/weight/chart'
};
```

That's it. 

### Calling AWS from application

***For Auth***

- Import Auth Library:
    - ```import { Auth } from 'aws-amplify';```

- Call appropriate methods:
    - For SignIn - Auth.signIn()
    - For Signup - Auth.signUp()
    - For Signout - Auth.signOut()

        Example: 

        ```javascript
        import { Injectable } from '@angular/core';

        import { Auth } from 'aws-amplify';
        import { EmailValidator } from '@angular/forms';

        @Injectable({
        providedIn: 'root'
        })
        export class AuthService {
        
        constructor() { }

        public async signIn(email: string, password: string) {
            try {
            const user = await Auth.signIn(email, password);
            localStorage.setItem('isAuthenticated', 'true');
            return true;
            } catch (error) {
            return false;
            }
        }

        public isUserAuthenticated(): boolean {
            let isAuthenticated = localStorage.getItem('isAuthenticated');
            if( isAuthenticated && isAuthenticated === 'true' ) {
            return true;
            }
            return false;
        }

        public async signout() {
            localStorage.removeItem('isAuthenticated');
            await Auth.signOut();
        }

        public async signUp(email: string, password: string) {
            try {
            const { user } = await Auth.signUp({
                username: email,
                password: password,
                attributes: {
                    email: email
                }
            });
            return true; 
            } catch (error) {
            console.log(error);
            return false;
            }
        }

        }
        ```

***For API calls***

- Import amplify by adding ```import { Auth, API } from 'aws-amplify';```
    - API library has options for various http verbs such as GET, POST, PUT, DELETE etc

            ```javascript
                private async loadData() {
                    let api_path = environment.api_version 
                                        + environment.get_babies_path;
                    let getData;
                    let initD = {

                    };
                    getData = await API.get(environment.api_name, api_path , initD);
                }
            ```
    - You can use init param to pass request headers, body etc.

        ```javascript
            let initD = {
                body: {
                    date: moment(date).format(this.DATE_FORMAT),
                    weight: parseFloat(weight)
                },
                headers: {
                    accept: 'application/json'
                }
            };
        ```

For an api that is exposed via API gateway which uses AWS_IAM auntentication, amplify REST API library will automatically sign the request via the Auth module. This will save lots of time and effort during development. If not, after authentication from cognito user pool, we have to get temporary credentials from cognito identity pool to sign the request and access AWS resources such as an API gateway. 

![cognito]({{ site.baseurl }}/assets/images/amplify-auth.png)


One more final thing is to disable the ```buildOptimiser``` by setting it to ```false``` in ```angular.json```.

