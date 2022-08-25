---
title: "AWS one punch"
date: 2021-12-19T10:21:44+13:00
draft: false
tags: ["AWS", "IAM Role Credentials", "Golang"]
readingTime: 8
customSummary: One punch to grant all command prompts AWS access with IAM role credentials in OSX.
---

## Background
\
There was a time when I was working on a feature which required changes to more than ten microservices. In the dev testing stage I got all the services up running in the cloud by applying [Cloudformation](https://aws.amazon.com/cloudformation/) changes from local. To simplify the work, I had one command prompt opened for each service so that I can build, package and deploy it whenever I changed something.
  
&nbsp;
## Pain points
\
[AWS SSO](https://aws.amazon.com/single-sign-on/) is used in our company to manage the AWS accounts as well as cloud resources, and we can obtain the [IAM temporary credentials](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtogetcredentials.html) in the user portal to access cloud resources from command lines. The pain point is the manual work (generate credentials, copy paste and execute in the command prompt, or to update the local credentials file) needs to be done every time when the temporary credentials are expired, and it will become worse when there are multiple command prompts opened like in my case.
  
&nbsp;
## Solution
\
So I started this project [AWS-one-punch](https://github.com/GaaraZhu/aws-one-punch) to automate this work. Now with only one command I can grant cloud access for all command prompts opened or will be opened.
  
&nbsp;
## Installation
&nbsp;
1. Install via Homebrew  
&nbsp;
```
brew install gaarazhu/aws-one-punch/aws-one-punch
```
  
&nbsp;  
&nbsp;
2. set the AWS user portal in `~/.bash_profile` or equivalent and reload it with `source ~/.bash_profile`  
&nbsp;
```
export AWS_CONSOLE_DOMAIN="garyz.awsapps.com"
```
  
&nbsp;  
&nbsp;
3. check the installation  
&nbsp;
![install-check](/images/aws-one-punch/install-check.png)
  
&nbsp;
## Usage  
&nbsp;
1. List all assigned AWS accounts
&nbsp;
```
$ aws-one-punch list-accounts
AccountId: ins-sd4312, accountName: 20890663 (MRP IaaS Prod)
AccountId: ins-2ssfds, accountName: 79300001 (Sandbox 1)
AccountId: ins-3sadfa, accountName: 69127290 (MRP IaaS Non-Prod)
AccountId: ins-siki23, accountName: 58868209 (Data Analytics)
AccountId: ins-14oasn, accountName: 66060440 (Shared Services)
```
  
&nbsp;  
&nbsp;
2. List all assigned AWS IAM role in an AWS account 
&nbsp;
```
$ aws-one-punch list-roles --account-id ins-3sadfa
RoleName: DigitalDeveloperNonprodAccess
```
  
&nbsp;  
&nbsp;
3. Grant all command promopts AWS access with temporary credentails from an IAM role
&nbsp;
```
$ aws-one-punch access --account-name 69127290 --role-name DigitalDeveloperNonprodAccess
AWS access granted with account 69127290 and IAM role DigitalDeveloperNonprodAccess
```
  
&nbsp;
### Conclusion
\
[aws-one-punch](https://github.com/GaaraZhu/aws-one-punch) automates the manual process to generate IAM role credentials and to grant command prompts AWS access with the credentials regardless of how many prompts there are. Hope this can help if you have the same problem.

For furthur simplification, we can create an alias for above one punch command, or we can have it managed somehow, for example with
[pet](https://github.com/knqyf263/pet).  
&nbsp;  
&nbsp;

