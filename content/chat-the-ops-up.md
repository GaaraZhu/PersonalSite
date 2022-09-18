---
title: "Chat the Ops up"
date: 2022-03-28T11:09:48+13:00
draft: false
tags: ["Microservices", "CI/CD", "ChatOps", "Atlassian Bamboo", "Microsoft Teams", "AWS", "Typescript", "Serverless"]
readingTime: 10
customSummary: This is the story of how we improved our CI/CD methodology by using Bamboo-on-Teams, a serverless ChatOps tool for interacting with Atlassian Bamboo from Microsoft Teams.  
---

## TL;DR
\
This is the story of how we improved our [CI/CD](https://en.wikipedia.org/wiki/CI/CD) methodology by using [Bamboo-on-Teams](https://github.com/GaaraZhu/bamboo-on-teams), a serverless [ChatOps](https://www.atlassian.com/blog/software-teams/what-is-chatops-adoption-guide) tool for interacting with Atlassian Bamboo from Microsoft Teams.  
&nbsp;

## Background
\
We are shifting from [monolith architecture](https://en.wikipedia.org/wiki/Monolithic_application) to [microservices](https://en.wikipedia.org/wiki/Microservices) for one of our systems. The goal is to break down the module dependencies, to make the system maintainable and scalable. One challenge we had was the increasing operational complexity as we now have around 50 microservices to build and to deploy whereas there used to be only one.  
&nbsp;

## SDLC
\
A quick review of how we are developing, testing and delivering multiple features in parallel.
1. Developers use feature branches to work on individual feature changes.
2. Building and deploying from the feature branch for dev testing, and create a pull request to merge the feature branch to the target release branch.
3. Merging the feature branch, building and deploying from the release branch for QA testing.
4. Deploying changes to UAT for final regression testing.
5. Production release.

![workflow](/images/chat-the-ops-up/cicd-workflow.png)
  
&nbsp;

## Pain points  
&nbsp;

### Manual intervention
\
Imagine that you are working on a feature which requires changes in **10** microservices. In Bamboo you will have to:
* create `20` branch plans
    - 10 branch plans for the feature branch
    - 10 branch plans for the release branch
* build `20` times
    - 10 builds for the feature branch plan
    - 10 builds for the release branch plan
* deploy `40` times
    - 10 deployments to Dev environment
    - 10 deployments to QA environment
    - 10 deployments to UAT environment
    - 10 deployments to Production environment

If bugs are found during the testing, more build and deployment job executions will be required.  
&nbsp;
### Status notification  
\
The deployment job can only be executed after the build job is finished. Although Bamboo has a good support of notifications to prevent us from checking the job status manually by refreshing the pages, it still requires manual configuration for each Bamboo job (Build and Deploy).  
&nbsp;
## Solution
\
To make our life easier, I created [Bamboo-on-Teams](https://github.com/GaaraZhu/bamboo-on-teams) with below main features to make Microsoft Teams the centralised place for Bamboo operations.
* Bamboo interaction - interact with Bamboo in Microsoft Teams with wide range of [commands](https://github.com/GaaraZhu/bamboo-on-teams#available-commands), and get Teams notifications afterwards. 
* On-demand CI/CD automation - automatically build, test, and deploy with Bamboo in Microsoft Teams.
* Hanging detection - detect hanging Bamboo jobs and alert in Microsoft Teams.
  
&nbsp;

## Demo 
&nbsp;
[![Demo](https://img.youtube.com/vi/JR8zbS7uKuA/0.jpg)](https://youtu.be/JR8zbS7uKuA)
  
&nbsp;

## Security controls  
&nbsp;
* Webhook security
  - [Outgoing webhook](https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-outgoing-webhook?tabs=urljsonpayload%2Cdotnet) can only interact with external web service, with the message text. No access to other APIs, like message history, channel information etc.
  - [Incoming webhook](https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook) can only receive JSON data with required schema for display, and has no access to other APIs, same with the outgoing webhook.
* [HMAC](https://en.wikipedia.org/wiki/HMAC) between Teams and Bamboo-on-Teams
* Dedicated Bamboo API token used for Bamboo-on-Teams
* Bamboo resource permission verification (view, execute etc)
* Production environment elimination and whitelisting (configurable)
  
&nbsp;

## Conclusion
\
In the journey of CI/CD, we are all learning and trying to improve the overall process, to make it more efficient and more collaborative. This is a tool that solved our problems and made our life easier. If you have similar problems with what we had, please consider this tool.

[Bamboo-on-Teams](https://github.com/GaaraZhu/bamboo-on-teams) is built on top of Typescript, Serverless and AWS tech stacks which you can find the details in the project page. Issues and PRs are welcome and encouraged: `Bamboo-on-Teams` is MIT licensed.  
&nbsp;  
&nbsp;
