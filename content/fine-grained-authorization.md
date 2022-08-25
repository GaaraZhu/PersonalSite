---
title: "Fine-grained authorization with Amazon Cognito user pool and API Gateway"
date: 2022-04-10T17:09:20+13:00
draft: false
tags: ["Microservices", "Authorization", "Serverless", "AWS", "Amazon Cognito", "API Gateway", "Serverless"]
readingTime: 10
customSummary: This post shows how we implemented a fine-grained system-to-system authorization for microservices application using Amazon Cognito user pool and API Gateway. 
---

Shifting from monolith architecture to microservices are full of challenges where one primary is how to manage security and access control properly. As each microservice is running indenpendly and communicates with each other to perform a specific function, we need to authenticate the incoming request to ensures that it comes from a legitimate service.

This post shows how we implemented a fine-grained system-to-system authorization for microservices application using [Amazon Cognito user pool](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html) and [API Gateway](https://docs.aws.amazon.com/apigateway/).
  
&nbsp;
## Solution
\
[Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html) is a powerful service for application authentication, authorization, and user management. The idea here is to use its user directory service [User Pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html) to authenticate calls between microservices as API Gateway is natively integrated with it to validate identities of consumer.

Architecture diagram:
![auth_flow](/images/fine-grained-authorization/auth_flow.png)
  
&nbsp;
## Key components
There are three key components here:
* **Cognito Resource server**: Each service has a dedicated resource server with pre-defined scopes for its resources(API method, Lambda etc)
* **Cognito App client**: Each service has a dedicated app client with limited scopes it needs to access external resources
* **API GW Cognito user pool authorizer**: Each API resource has an Cognito authorizer configured for authenciation and authorization
  
&nbsp;
### Cognito Resource server
\
[Resource server](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html) is what we can use to manage the scopes that are required for accessing our REST resources. For example, we have scope `customer.read` from resouce server `customers-v1-resource-server` for resource `/v1/customers/{customerId}` in service `customers-v1`, indicating that to access the resource a JWT token with this scope is required in the request.

Resource server definition:
```yaml
  CognitoUserPoolResourceServer:
    Type: 'AWS::Cognito::UserPoolResourceServer'
    DeletionPolicy: Retain
    Properties:
      Identifier: 'customers-v1-resource-server'
      Name: 'customers-v1-resource-server'
      Scopes:
        - ScopeDescription: 'Customer Read Scope' # to be used in below method resource
          ScopeName: 'customer.read'
      UserPoolId: '1ds23esfa'
```
API method resource definition:
```yaml
 CustomerGet:
    Type: 'AWS::ApiGateway::Method'
    Properties:
        HttpMethod: GET
        RestApiId: !Ref: TestApi
        ResourceId: !Ref: CustomerResource # /v1/customers/{customerId}
        AuthorizationType: COGNITO_USER_POOLS
        AuthorizerId: !Ref: CognitoUserPoolAuthorizer # defined below
        AuthorizationScopes:
            - 'customers-v1-resource-server/customer.read' # scope from resource server above
        Integration: ...
```
  
&nbsp;
### Cognito App client
\
[App client](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-client-apps.html) is where we define the auth flow, and where we specify the allowed scopes for the current service to access external resources. for example, we have app client `accounts-v1-app-client` with allowed scope `customers-v1-resource-server/customer.read`(a resource server and scope combination from the target service), indicating that `accounts-v1` service can only access resource `/v1/customers/{customerId}` in service `customers-v1`.

```yaml
CognitoUserPoolClient:
    Type: 'AWS::Cognito::UserPoolClient'
    UpdateReplacePolicy: Delete
    DeletionPolicy: Delete
    Properties:
        AllowedOAuthFlows:
            - 'client_credentials'
        AllowedOAuthScopes:
            - 'customers-v1-resource-server/customer.read' # allowed custom scopes
        AllowedOAuthFlowsUserPoolClient: true
        ClientName: 'accounts-v1-app-client'
        ExplicitAuthFlows:
        - ALLOW_REFRESH_TOKEN_AUTH
        GenerateSecret: true
        UserPoolId: 1ds23esfa
```
  
&nbsp;
### API GW Cognito user pool authorizer
\
When receiving an API call request, the authorizer will authenticate the request by checking whether an access token is supplied and whether it's valid, after that it will authorize the call based on the specified custom scopes for the access-protected resources. For example, a request with access token having `customers-v1-resource-server/customer.read` scope will pass the authentication and will be authorized to access resource `/v1/customers/{customerId}` in service `customers-v1`.
```yaml
CognitoUserPoolAuthorizer:
    Type: 'AWS::ApiGateway::Authorizer'
    Properties:
    AuthorizerResultTtlInSeconds: 300
    IdentitySource: 'method.request.header.Authorization'
    Name: 'cognito-user-pool-authorizer'
    RestApiId: !Ref ApiGatewayRestApi
    Type: COGNITO_USER_POOLS
    ProviderARNs:
        - 'arn:aws:cognito-idp:4234234234:324423523525:userpool/1ds23esfa'
```


&nbsp;
## Challenges
1. Deployment dependency

This pattern introduces a service dependency which requires target service resources and scopes to be created/updated first prior to the consumer service. For example, service `accounts-v1` can not be deployed unless the relied scope has been created by deploying service `customers-v1`.

**Solution:** use [release](https://github.com/GaaraZhu/bamboo-on-teams#release) command in [Bamboo-on-Teams](https://github.com/GaaraZhu/bamboo-on-teams) to deploy services in sequential batches
  
&nbsp;
2. App client credentials management

The service-to-service interaction starts with a user pool sign-in with the app client credentials and a JWT token will be returned from Cognito to the initiator for external resource access. We are creating the secure parameters in Amazon SSM manually to store the client credentials for each microservice after the deployment. With the increasing number of µservices, we need a tool to do this securely and automatically for us.

**Solution:** https://www.serverless.com/plugins/serverless-app-client-credentials-to-ssm
  
&nbsp;  
## Conclusion
\
With this fine-grained access control mechanism, we now can guarantee that only authenticated services will be allowed to access the resource, and only resources with specified scopes can be accessed by the service.
  
&nbsp;  
&nbsp;