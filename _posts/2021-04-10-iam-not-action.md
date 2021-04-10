---
layout: post
title:  "IAM: NotAction in Policy"
author: VJ
categories: [ AWS ]
tags: [AWS, IAM, Policy, Security]
image: assets/images/iam.png
description: "Write better IAM policies"
featured: true
hidden: false
comments: true
---

One of the main thing that you have to get it right when you move into cloud is your Identity & Access Management. Through IAM you define who the users(humans and computers) are  and also what they can access and how free they are to perform operations on various AWS resources.

Polcies in AWS help you to define the access and their limits for an IAM user or resource to operate on a particular AWS resource or a set of AWS resources.

There are various policy types that are used for various purposes. Lets quickly refresh them :

- **Identity based policies** : They can be attached to a IAM users or groups to grant access on various AWS resources defined in the policy.
- **Resource based ppolicies** : These are attached inline policies for a resource. Example S3 bucket policy. Since this is attached to a resource, you need to specify the 'Principal' to which this is applicable. 
- **Permission boundaries** : This specifies the maximum permissions that an identity base policy can grant to an IAM entity. Once this is attached to an IAM entity, they can only perform the actions allowed by IAM policies and its permission boundaries.
- **SCPs or Service Control Policies**: These are guardrails applied at the Organization or at an OU(Organization) level. This limis the permissions for the entities in the accounts under the organization.
- **Access Control Lists (ACLs)** : Used to control which principals in another account can access a resource. Not all AWS resouces support ACLs. Example of a resource supporting ACLs is S3.
- **Session Policies** : This is something which you pass a parameter when we create a temporary session for a role or federated user via sts. We can pass session policies programatically with the ```AssumeRole```, ```AssumeRoleWithSAML``` or ```AssumeRoleWithWebIdentity``` API operations.


This post in mainly intended to provide a brief idea about ``NotAction`` in an IAM policy.



Most of the policies that we write consist of JSON elements: 
 - Principal
 - Action
 - Resource
 - Condition
 - Effect

One way of remembering this is with the word PARC (I got this from one of the AWS summit events). I think of Parking an electric vehicle (PARCe - e is for electric or effect).

Now consider this scenario where you need to allow your users to access all AWS resouces except, IAM. But they should be free to list, generate and delete their own access keys in IAM. How could we achieve this?

Since we know that we should not allow user to access IAM, we would think that we could deny the user IAM permissions with a ```Deny``` statement:


```JSON
{
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "iam:*"
            ],
            "Resource": "*"
        }
    ]
}
```

With this above policy we have achieved on thing, that is we have denied the access to IAM for the entities to which this policy is atttached. Now we need to allow users to access all other AWS resources. This is due to the fact that if we do not have an explicit allow of resources, by default it will be denied. So we add on to the above policy as below



```JSON
{
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "iam:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

Now we have allowed the enitities with the new ```Allow``` statement to access all other resources. 

Till now we are good, but not finished. We need to do one more thing, which is to allow our users to manage their own access keys in IAM. Since this is an allow of actions on IAM, we might first think of adding some explict ```Allow``` statements like :


```JSON
{
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "iam:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        },
        {
            "Effect": "Allow"
            "Action": [
                "iam:CreateAccessKey",
                "iam:DeleteAccessKey",
                "iam:ListAccessKeys",
                "iam:UpdateAccessKey"
            ],
            "Resource": "arn:aws:iam::*:user/${aws:username}"
        }
    ]
}
```

Now lets think, will this work? We have added explicit ```Allow``` statement in the policy for Access Keys management. However this won't work as we already have an explicit ```Deny``` for IAM actions. Hence ```Deny``` is the one which will be taking precedence. So, despite having allows for AccessKey management, user will be denied permission.

> **Note**:  Explicit Deny always have precedence.


Now, how can we achieve what we want? For this scenario we could use ```NotAction``` of IAM policy. We could rewrite the policy as 

```JSON
{
    "Statement": [
        {
            "Effect": "Allow",
            "NotAction": [
                "iam:*"
            ],
            "Resource": "*"
        }
    ]
}

```

This will allow entities attached with this policy, previleges to access everything, except all actions of IAM. So we essentially completed our first two tasks. We blocked user from performing IAM actions via ```NotAction``` and also allowed all other actions on all other resources. Now the final thing that is left is to allow entities to manage their own ```AccessKeys```. For this we can modify the policy as below:


```JSON
{
    "Statement": [
        {
            "Effect": "Allow",
            "NotAction": [
                "iam:*"
            ],
            "Resource": "*"
        }, {
            "Effect": "Allow",
            "Action": [
                "iam:CreateAccessKey",
                "iam:DeleteAccessKey",
                "iam:ListAccessKeys",
                "iam:UpdateAccessKey"
            ],
            "Resource": "arn:aws:iam::*:user/${aws:username}"
        }
    ]
}

```


By this we are allowing users to manage their own access credentials via IAM. Since we are not explicitly denying in the first statement, this allow will be honoured.

I hope you understood the use of ```NotAction``` in an IAM policy. It will allow every actions, except the one mentioned in the ```NotAction```. But keep in mind, this is not an ```explict deny```.

> And remember EXPLICIT DENY WILL ALWAYS WIN.

