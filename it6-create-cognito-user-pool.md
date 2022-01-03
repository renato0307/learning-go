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

## Generate a JSON Web Token (JWT)

```sh
export AUTH_TOKEN=`echo $CLIENT_ID:$CLIENT_SECRET | base64`
export AUTH_HEADER="Basic $AUTH_TOKEN"
```

```sh
export TOKEN_ENDPOINT=https://$DOMAIN.auth.$AWS_REGION.amazoncognito.com
```

```sh
http POST $TOKEN_ENDPOINT/oauth2/token Authorization:$AUTH_HEADER
```

