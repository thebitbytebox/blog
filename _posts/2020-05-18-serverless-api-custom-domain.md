---
layout: post
title:  "Serverless APIs with custom domain"
author: VJ
categories: [ Serverless ]
tags: [REST, API, serverless, AWS, lambda, api gateway, route53]
image: assets/images/route-53.png
description: "A custom domain for you API gateway"
featured: true
hidden: true
comments: true
---

When we create an API gateway to front our lambda functions, we normally get an AWS provided domain which can be used to invoke our APIs.
AWS provided domain for our API gateway will look something like this ``` https://gbg642gdfg5.execute-api.ap-south-1.amazonaws.com/<stage>/<path>```. 


This domain is not beautiful, I mean how does one remember this?

We could use a custom domain to overcome this. 

For this, we need to have a domain name. You can purchase one from amazon or from outside through an ICANN-accredited registrar like namecheap, godaddy etc. You can then use AWS DNS service by adding it to the hosted zone in route53.

### Create a hosted zone for the domain

To create a hosted zone, go to route53 > hosted zone > create hosted zone

Under the domain name, add your root domain, e.g. ```thebitbytebox.com```. You can choose either a public or private hosted zone. 

A public hosted zone  determines how traffic is routed on the internet. while a private hosted zone determines how traffic is routed within Amazon VPC.

Let's choose public and hit create.

![Hosted Zone]({{ site.baseurl }}/assets/images/hosted zone- create.png)


This should create some name server(NS) records. If your domain is from an external source, take these NS records and add it under the DNS management of your registrar. This would take sometime to get resolved across the internet. For me it took approximately around 50 minutes.

***Note***: do not add the . at the end of the NS records.

![Hosted Zones]({{ site.baseurl }}/assets/images/hostedzone-r53.png)

### SSL Certificate for the custom domain

Now, we can provision our ssl certificate using the AWS Certificate Manger. To do this, make sure you are on N. Virginia region as most of the AWS resources like cloud front requires your certificate to be in us-east-1 region.


To provision a public certificate, 

Open AWS Certificate Manager console and select ```Get started``` under ```Provision Certificates```. Select ```Request a public certificate``` radio button and subimit the request. This will open up a page asking to add domain names. You can add your domain name for e.g.  ```domain.com``` . You can also add your subdomains like ```api.domain.com```by clicking the ```Add another name to this certificate``` button or enter wildcard like ```*.domain.com``` to have a _subdomain if needed.



Now we need to validate the request by confirming the ownership of the domain. There are two methods, 

- DNS Validation - Choose this if you can modify the DNS configuration.
- Email Validation 

Continuing will take you to the review screen, where we can confirm the request. This will lead to the final step of Validation. Once validation  is complete, you can see the validation status changing from ```Validation complete``` from ```Pending Validation```.

_***Note***: If you have chosen DNS validation, you can expand  the DNS names under the validation screen. There, you can find the details of adding the CNAME to the DNS configuration of your domain. Add the ```CNAME``` in route53 under your domain name in the hosted zone._

### Adding Custom Domain to the API Gateway

In the API Gateway console, on the left hand side there is an option of CUstom domain names which can help you to to manually create the domain names. This will help in choosing the domain name and also its configurations like endpoint type, TLS, certificate etc.

After creating custom domain name in the API gateway, you can add a API mappings to map the API to your domain. You can choose to which stage you need this domain name. This will essentially route your request from your custom domain into the API gateway.

You can do the same using the serverless framework.

### Serverless API Gateway Custom Domain Mapping


You need to install ```serverless-domain-manager``` plugin

Run  
```
npm install serverless-domain-manager --save-dev
```

Include the plugin in your ```serverless.yml```

```yml
plugins:
  - serverless-domain-manager
```

Then add customDomain under the custom tag in yml

```yml
  customDomain:
    domainName: api.thebitbytebox.com
    stage: ${self:provider.stage}
    certificateName: thebitbytebox.com
    createRoute53Record: true
```

Save and run ```sls create_domain``` to create the custom domain under API gateway. It should take some time for AWS to provision a CLoudFront distribution. Once done, you can do an sls deploy. This will make a mapping of your API to the new custom domain based on the stage.

Now you can invoke the API gateway with a much nicer and beautiful domain names.