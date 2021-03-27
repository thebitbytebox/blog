---
layout: post
title:  "Compromised Access Key? Deal with it."
author: VJ
categories: [ AWS  ]
tags: [AWS, IAM]
image: assets/images/iam.png
description: "How to deal with a compromised Access Key"
featured: true
hidden: true
comments: true
---

When it comes to development in AWS, most of the organizations create access keys for its developers and other users. Some organizations uses access keys in their production and dev EC2 instanes. There could be situations when these keys gets misplaced (such as gets committed in public repo, sent via email etc) and gets into the hands of wrong people. It is even more dangerous if the compromised user access key has full admin previleges. 

> What can we do when an access key gets compromised? 

Most of us first think of deleting the access key. But does that solve the potential problem at hand? 

Yes, deleting compromised accesskey is a part of the solution but it won't solve the whole menance.

Deleting or making the access key will definitely stop the unauthorized user from using it anymore. But won't stop the attacker, if she/he has already generated temporary security credentials. If so he/she will have all the access that the user has despite the deletion of the access key. 


#### Analyze
Before you worry and delete the compromised key, analyze the previleges that the key has. Sometimes it might not be that damaging to deal with it in a priority. Analyze the implications of deleting/disabling key on your productions systems. Understand the possibility of a downtime and whether you could afford it or not. If the compromised key could potentially impact your sensitive data and systems, it is better to disable the key immediately in order to avoid sensitive data leaks and then deal with downtime later.


#### Disable Key
After all these analysis, you can go ahead and disable the key as a best practise. This is because, if the whole hell break loose on your critical systems, you can make it active to avoid service disruption.

#### Deal with Temporary Credentials

By this time if the unauthorized user has generated temporary security credential, above steps wont stop the user from getting his/her hands dirty. Attacker has 15 minutes to 36 hours until the token expires. To deal with it, we would require to disable the permissions for the credentials. There is no explicit way to disable a temp security credentials. The possible ways to deal with it is to remove the user or to add an explicit deny. Since permissions are evaluated each time an AWS request is made using the credentials, you can achieve the effect of revoking the credentials by changing the permissions for the credentials even after they have been issued. If the credential is generated with AssumeRole or AssumeRoleWithSAML, revoke the previleges for the assumed role. If it is for a federated token, revoke the permission of the federated user. This will stop the malicious user from abusing your systems.

#### Impact Analysis

Once you have dealt with the temporary security credentials, understand if the attacker has done anything bad or not. Example if he had admin previleges, he could create other users and roles who have admin previleges and can continue to exploit your systems. You can use tools such as Cloudtrail, Athena etc. for your analysis. Based on analysis you can deal with it by deleting the unwanted users, roles etc.

#### Document

Now that we have cleaned up our system from intrusion, we can restore access and also document it.

#### Conclusion
As you see, giving more previleges to a user than they require could be more damaging. Hence always give the least amount of previleges that they require. Make sure you decentalize your logs to keep it safe from attackers. This will help us in doing a full scale analaysis of damage. Try to use roles rather than access keys when possible.

Let me know your thoughts.