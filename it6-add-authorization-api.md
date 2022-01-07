# Add authorization to the API

_Authentication_ is the process of proving that you are who you say you are.

_Authorization_ is the act of granting an authenticated party permission to do
something. It specifies what data you're allowed to access and what you can do
with that data.

We are going to support authorization using OAuth2 scopes.

As described in the
[OAuth 2.0 specs](https://datatracker.ietf.org/doc/html/rfc6749#section-3.3):

> The authorization and token endpoints allow the client to specify the
> scope of the access request using the "scope" request parameter.  In
> turn, the authorization server uses the "scope" response parameter to
> inform the client of the scope of the access token issued.

So we are going to create a scope per function. For the client to be able to
have access to a function, it needs to have a JWT with that scope inside.

Our scopes will match the functions available in the API:

* https://$RESOURCE_SERVER_NAME.com/programming-uuid
* https://$RESOURCE_SERVER_NAME.com/programming-jwtdebugger
* https://$RESOURCE_SERVER_NAME.com/finance-currconv


To fully support this we need to:

1. Configure new scopes
1. Allow the client to use the new scopes
1. Implement a Gin middleware to verify the existence of the right scope when
executing a function

## Configuring new scopes

We will use the AWS CLI to create the scopes.

Don't forget to define the environment variables used below, using the
instructions described in 
[Configure Amazon Cognito as an OAuth2 server](it6-create-cognito-user-pool.md):

```sh
aws cognito-idp update-resource-server \
    --user-pool-id $POOL_ID \
    --identifier https://$RESOURCE_SERVER_NAME.com \
    --name $RESOURCE_SERVER_NAME \
    --scopes ScopeName="programming-uuid",ScopeDescription="Access to execute programming/uuid" \
             ScopeName="programming-jwtdebugger",ScopeDescription="Access to execute programming/jwtdebugger" \
             ScopeName="finance-currconv",ScopeDescription="Access to execute finance/currconv" \
    --profile $AWS_PROFILE \
    --region $AWS_REGION
```

## Allow the client to use the new scopes

To allow the client to use the scopes defined we need to execute the following
command:

```sh
aws cognito-idp update-user-pool-client \
    --user-pool-id $POOL_ID  \
    --client-id $CLIENT_ID \
    --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_ADMIN_USER_PASSWORD_AUTH" "ALLOW_REFRESH_TOKEN_AUTH" \
    --supported-identity-providers "COGNITO" \
    --allowed-o-auth-flows "client_credentials" \
    --allowed-o-auth-flows-user-pool-client \
    --allowed-o-auth-scopes "https://$RESOURCE_SERVER_NAME.com/programming-uuid" \
                            "https://$RESOURCE_SERVER_NAME.com/programming-jwtdebugger" \
                            "https://$RESOURCE_SERVER_NAME.com/finance-currconv" \
    --profile $AWS_PROFILE \
    --region $AWS_REGION
```

If we generate a new token using the following command:

```sh
http POST $TOKEN_ENDPOINT \
    Authorization:$AUTH_HEADER \
    Content-Type:application/x-www-form-urlencoded \
    --raw "grant_type=client_credentials&scope="
```

And we inspect the contents of the JWT we can see in the payload something like:

```json
{
  "sub": "...",
  "token_use": "access",
  "scope": "https://learninggolang.com/programming-uuid https://learninggolang.com/finance-currconv https://learninggolang.com/programming-jwtdebugger",
  "auth_time": 1641582218,
  "iss": "...",
  "exp": 1641585818,
  "iat": 1641582218,
  "version": 2,
  "jti": "...",
  "client_id": "..."
}
```

The `scope` attributes contains all the scopes the client has access to.

## Implement a Gin middleware to handle authorization

WIP

## Unit testing

WIP

## Manual testing

WIP

## Wrap up

WIP