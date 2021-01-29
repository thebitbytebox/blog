---
layout: post
title:  "Compromised Access Key? Deal with it."
author: VJ
categories: [ AWS, IAM, Security ]
tags: [AWS, IAM]
image: assets/images/iam.png
description: "How to deal with a compromised Access Key"
featured: true
hidden: true
comments: true
---

When it comes to development in AWS, most of the organizations create access keys for its developers and other users. Some organizations uses access keys in their production and dev EC2 instanes. There could be situations when these keys gets misplaced (such as gets committed in public repo, sent via email etc) and gets into the hands of wrong people. It is even more dangerous if the compromised user access key has full admin previleges. 

> What can we do when an access key gets compromised? 

Most of us first think of deleting the access key. But does that solve the potential problem at hand. Yes, deleting compromised accesskey is a part of the solution but it won't solve the whole menance.

Deleting or making the access key will definitely stop the unauthorized user from using it anymore. But wont stop him if, he has generated temporary security credentials. He will have all the access the user has despite the deletion of the access key. 


#### Analyze
Before you worry and delete the compromised key, analyze the previleges the key has. Sometimes it might not be that damaging to deal with in priority. Analyze the implications of deleting/disabling key on your productions system. Understand the possibility of a downtime and whether you could afford it or not. If the compromised key could potentially impact your sensitive data and system, it is better to disable the key immediately to avoid sensitive data leaks and deal with downtime later.


#### Disable Key
After all these analysis, you can go ahead and disable the key as a best practise. This is because if the whole hell break loose on your critical system, you can make it active again to avoid service disruption.

#### Deal with Temporary Credentials

By this time if the unauthorized user has generated temporary session token, above steps wont stop the user from getting his/her hands dirty. He has 15 minutes to 36 hours until the token expires. To deal with it we would require to disable permission for the credentials. There is no explicit way to disable a temp session token. Hence, the possible ways to deal with it is to remove the user or to add an explicit deny. Since permissions are evaluated each time an AWS request is made using the credentials, you can achieve the effect of revoking the credentials by changing the permissions for the credentials even after they have been issued. If the token is generted with AssumeRole or AssumeRoleWithSAML, revoke the previleges for the assumed role. If it is for a federated token, revoke the permission of the federated user. This will stop the malicious user from abusing your access key.

#### Impact Analysis

Once you have dealt with the temporary session tokens, understand if the attacker has done anything bad or not. Example if he has admin previleges, he could create other users and roles who have admin previleges and can continue to exploit your system. You can use tools such as cloudtrail, Athena etc. for your analysis. Based on analysis you can deal it with deleting the unwanted users, roles etc.

#### Document

Now that we have cleaned our system from intrusion, we can restore access and also document it.

#### Conclusion
As you see, giving more previleges to a user than they require could be more damaging. Hence always give the least amount of previleges that they require. Make sure you decentalize your logs to keep it safe from attackers. This will help us in doing a full scale analaysis of damage. Try to use roles rather than access keys when possible.