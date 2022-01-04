# Configure Amazon Cognito as an OAuth2 server

Building a OAuth2 server is a massive task. So we will not do it in this scope.

There are some open source alternatives and proprietary that we can use for free
for our purposes. 

In this context we are going to use [Amazon Cognito](https://aws.amazon.com/cognito).

To be able to execute the Cognito configurations we will need:

1. AWS Account (create one at [aws.amazon.com](https://aws.amazon.com))
1. Access key id and secret access key for programmatic calls to AWS (check 
how to do it in the
[documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey))
1. AWS CLI (check how to install it [here](https://aws.amazon.com/cli))
1. A command-line JSON processor (check how to install it [here](https://stedolan.github.io/jq))


First, define the following environment variables:

```sh
export AWS_PROFILE=default # or the profile defined when configuring the CLI
export AWS_REGION=eu-west-1 # you can pick another region
export POOL_NAME=learning-go-pool
export DOMAIN=learning-go-renato0307 # pick a unique name
export RESOURCE_SERVER_NAME=learning-go-api
```

## Create a User Pool

From the AWS documentation:

> A user pool is a user directory in Amazon Cognito. With a user pool, your
> users can sign in to your web or mobile app through Amazon Cognito.
> (...)
> After successfully authenticating a user, Amazon Cognito issues JSON web 
> tokens (JWT) that you can use to secure and authorize access to your own APIs. 

To create a Cognito User Pool execute:

```sh
aws cognito-idp create-user-pool \
    --pool-name $POOL_NAME \
    --profile $AWS_PROFILE \
    --region $AWS_REGION
```

Get the pool id and save it in the `POOL_ID` environment variable:

```sh
export POOL_ID=`aws cognito-idp list-user-pools \
    --profile $AWS_PROFILE \
    --region $AWS_REGION \
    --max-results 1 \
    --region eu-west-1 | jq -r ".UserPools[0].Id"`
```

## Create a Domain

From the AWS documentation:

> You can configure the address of your sign-up and sign-in webpages. 
> You can use an Amazon Cognito hosted domain and choose an available domain 
> prefix, or you can use your own web address as a custom domain.

To create a domain execute: 

```sh
aws cognito-idp create-user-pool-domain \
    --user-pool-id $POOL_ID \
    --domain $DOMAIN \
    --profile $AWS_PROFILE \
    --region $AWS_REGION 
```

## Create a Resource Server

From the AWS documentation: 

> When using a REST API that requires OAuth 2.0 access tokens, you can use 
> Amazon Cognito to issues those tokens and allow users to access resources. 
> In this environment, the REST API serves as your resource server.

To define a resource server execute:

```sh
aws cognito-idp create-resource-server \
    --user-pool-id $POOL_ID \
    --identifier https://$RESOURCE_SERVER_NAME.com \
    --name $RESOURCE_SERVER_NAME \ 
    --profile $AWS_PROFILE \
    --region $AWS_REGION
```

We will want to have authorization claims inside our JWTs. These claims are
going to be used to check if the user has permissions access each of the API
operations (e.g. using the currency converter).

To achieve that we need to define resource server scopes. That can be achieved
by updating the resource server:

```sh
aws cognito-idp update-resource-server \
    --user-pool-id $POOL_ID \
    --identifier https://$RESOURCE_SERVER_NAME.com \
    --name $RESOURCE_SERVER_NAME \
    --scopes ScopeName="all",ScopeDescription="Access to all resources" \
    --profile $AWS_PROFILE \
    --region $AWS_REGION
```

In this case, and for now, we are creating only one scope.

## Create an App Client

From the AWS documentation:

> An app client connects your app to the user pool and authorizes Amazon Cognito
> to generate OAuth 2.0 tokens. App clients grant an app access to your users' 
> data and attributes. Your apps will then use this data to authenticate and
> authorize access to your resources.

To create an app client execute:

```sh
aws cognito-idp create-user-pool-client \
    --user-pool-id $POOL_ID  \
    --client-name learning-go-api-client-1 \
    --generate-secret \
    --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_ADMIN_USER_PASSWORD_AUTH" "ALLOW_REFRESH_TOKEN_AUTH" \
    --allowed-o-auth-flows "client_credentials" \
    --supported-identity-providers "COGNITO" \
    --allowed-o-auth-flows-user-pool-client \
    --profile $AWS_PROFILE \
    --region $AWS_REGION
```

To get the client id execute:

```sh
export CLIENT_ID=`aws cognito-idp list-user-pool-clients §
    --user-pool-id $POOL_ID \  
    --profile $AWS_PROFILE \ 
    --region $AWS_REGION | jq -r ".UserPoolClients[0].ClientId"`
```

To get the client secret execute:

```sh
export CLIENT_SECRET=`aws cognito-idp describe-user-pool-client \
    --user-pool-id $POOL_ID \
    --client-id $CLIENT_ID \
    --profile $AWS_PROFILE \
    --region $AWS_REGION | jq -r ".UserPoolClient.ClientSecret"`
```

To allow the client to call the APIs, we need to associate the resource server
scopes to the client. We could have done this while creating the user, but I
want to show you the command to be used when you need to add more.

```sh
aws cognito-idp update-user-pool-client \
    --user-pool-id $POOL_ID  \
    --client-id $CLIENT_ID \
    --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_ADMIN_USER_PASSWORD_AUTH" "ALLOW_REFRESH_TOKEN_AUTH" \
    --supported-identity-providers "COGNITO" \
    --allowed-o-auth-flows "client_credentials" \
    --allowed-o-auth-flows-user-pool-client \
    --allowed-o-auth-scopes "https://learning-go-api.com/all" \ # you can add more scopes here
    --profile $AWS_PROFILE \
    --region $AWS_REGION
```


## Generate a JSON Web Token (JWT)

Generate the authentication header to use:

```sh
export AUTH_TOKEN=`echo -n $CLIENT_ID:$CLIENT_SECRET | base64`
export AUTH_HEADER="Basic $AUTH_TOKEN"
```

Define the endpoint used to generate the tokens:

```sh
export TOKEN_ENDPOINT=https://$DOMAIN.auth.$AWS_REGION.amazoncognito.com/oauth2/token
```

Call the endpoint using `httpie` to get the token:

```sh
http POST $TOKEN_ENDPOINT  \
    Authorization:$AUTH_HEADER \
    Content-Type:application/x-www-form-urlencoded \
    --raw "grant_type=client_credentials&scope=https://learning-go-api.com/all"
```

The result should be similar to:

```json
HTTP/1.1 200 OK
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Connection: keep-alive
Content-Type: application/json;charset=UTF-8
Date: Tue, 04 Jan 2022 07:21:45 GMT
Expires: 0
Pragma: no-cache
Server: Server
Set-Cookie: XSRF-TOKEN=5f8f0345-139a-4e3c-bbce-eb12dabb300c; Path=/; Secure; HttpOnly; SameSite=Lax
Strict-Transport-Security: max-age=31536000 ; includeSubDomains
Transfer-Encoding: chunked
X-Application-Context: application:prod:8443
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
x-amz-cognito-request-id: 22f9a72b-a3bc-4d37-adf7-5d5743a44959

{
    "access_token": "eyJraWQiOiJrb1wvR2owY1JrdFwvd2hQbm9NeU00YTB5UWVRQ0p (...)",
    "expires_in": 3600,
    "token_type": "Bearer"
}
```
