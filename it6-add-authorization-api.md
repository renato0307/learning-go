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

For simplicity, we are going to implement the scopes verification by using a
convention over configuration approach.

By naming the scopes in a very similar way we build the URLs, we can easily code
an algorithm that does not require any additional setup.

This might not be the most appropriate solution for all scenarios but for our
current example is good enough.

For example, to access the `/v1/programming/uuid` URL the client needs to have
the `https://learninggolang.com/programming-uuid` scope. We just need to:
1. Pick the URL
1. Remove the version
1. Convert `/` into `-`

Moving the code, we first need to change the authentication middleware to
include the scopes in the Gin context, so the authorization middleware does
not need to parse the JWT again:

```go
func Authenticator(ac *AuthenticatorConfig) gin.HandlerFunc {
    return func(c *gin.Context) {

        // ...

        // Puts the scopes in the Gin context
        scope, _ := token.Get(ScopeKey)  // new
        c.Set(ScopeKey, scope)           // new
    }
}
```

Next we are going to add a new middleware named `Authorizer`.

To following files are needed:

```sh
touch internal/middleware/authorizer.go
touch internal/middleware/authorizer_test.go
```

The authorizer code follows. With the comments, it should be easy to understand.

```go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "github.com/renato0307/learning-go-api/internal/apierror"
    "github.com/rs/zerolog/log"
)

func Authorizer() gin.HandlerFunc {
    return func(c *gin.Context) {

        // extracts the required scope from the URL
        p1 := c.Request.URL.Path               // p1 = "/v1/programming/uuid"
        p2 := strings.ReplaceAll(p1, "/", "-") // p2 = "-v1-programming-uuid"
        p3 := strings.Split(p2, "-")           // p3 = ["", "v1", "programming", "uuid"]
        p4 := p3[2:]                           // p4 = ["programming", "uuid"]
        scope := strings.Join(p4, "-")         // scope = "programming-uuid"
        log.Debug().Msgf("scope for url is %s", scope)

        // gets the scopes added by the Authenticator middleware
        clientScopes := c.GetString(ScopeKey)
        clientScopesList := strings.Split(clientScopes, " ")
        log.Debug().Msgf("client scope list is %s", clientScopes)

        // tries to find a matching scope
        found := false
        for _, clientScope := range clientScopesList {
            found = strings.HasSuffix(clientScope, scope)
            if found {
                break
            }
        }

        // returns forbidden (HTTP status 403) if no valid scope is found
        if !found {
            log.Debug().Msg("no scope found for current route")
            c.AbortWithStatusJSON(
                http.StatusForbidden,
                apierror.New("Forbidden"))
            return
        }

        log.Debug().Msg("valid client scope found for current route")
    }
}
```

## Unit testing

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

The contents of the `authorizer_test.go` file follows. We are using the 
technique we used before, having a list of test cases and a for loop to execute
them all, keeping the code simpler.

```go
package middleware

import (
    "net/http"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/renato0307/learning-go-api/internal/apitesting"
    "github.com/stretchr/testify/assert"
)

func TestAuthorizer(t *testing.T) {

    testCases := []struct {
        Scopes     string
        URL        string
        RequestURL string
        Purpose    string
        StatusCode int
    }{
        {
            Scopes:     "https://learninggolang.com/programming-jwtdebugger",
            URL:        "/v1/programming-jwtdebugger",
            RequestURL: "/v1/programming-jwtdebugger?abcd",
            Purpose:    "scopes match URL",
            StatusCode: http.StatusOK,
        },
        {
            Scopes:     "https://learninggolang.com/programming-uuid",
            URL:        "/v1/programming-jwtdebugger",
            RequestURL: "/v1/programming-jwtdebugger",
            Purpose:    "scopes do not match URL",
            StatusCode: http.StatusForbidden,
        },
        {
            Scopes:     "",
            URL:        "/v1/programming-jwtdebugger",
            RequestURL: "/v1/programming-jwtdebugger",
            Purpose:    "no scopes defined",
            StatusCode: http.StatusForbidden,
        },
        {
            Scopes:     "https://learninggolang.com/programming-jwtdebugger https://learninggolang.com/programming-uuid",
            URL:        "/v1/programming-jwtdebugger",
            RequestURL: "/v1/programming-jwtdebugger",
            Purpose:    "list of scopes",
            StatusCode: http.StatusOK,
        },
    }

    for _, tc := range testCases {
        // arrange - init gin to use the middleware
        r := gin.New()
        r.Use(func(c *gin.Context) { // fake Authenticator
            c.Set(ScopeKey, tc.Scopes)
        })
        r.Use(Authorizer())
        r.Use(gin.Recovery())

        // arrange - set the routes
        r.POST(tc.URL, func(c *gin.Context) {})

        // act
        w := apitesting.PerformRequest(r, "POST", tc.RequestURL)

        // assert
        assert.Equal(t, tc.StatusCode, w.Code, tc.Purpose)
    }
}
```

## Changes in the main.go file

We need to make Gin use the new middleware:

```go
func main() {
    // Initialize Gin
    gin.SetMode(gin.ReleaseMode)
    r := gin.New()
    r.Use(middleware.DefaultStructuredLogger())
    r.Use(middleware.Authenticator(newAuthenticatorConfig()))
    r.Use(middleware.Authorizer()) // new
    r.Use(gin.Recovery())

    // ...
}
```

## Manual testing

Start the Gin server like we did before.

Request a token with all scopes:

```sh
http POST $TOKEN_ENDPOINT \
    Authorization:$AUTH_HEADER \
    Content-Type:application/x-www-form-urlencoded \
    --raw "grant_type=client_credentials&scope="
```

Make a request:

```sh
http POST localhost:8080/v1/programming/uuid Authentication:eyJraWQiOiJrb1wvR2owY1...
```

The result should be similar to:

```json
HTTP/1.1 200 OK
Content-Length: 47
Content-Type: application/json; charset=utf-8
Date: Fri, 07 Jan 2022 20:31:09 GMT

{
    "uuid": "5aae6940-d3f2-402e-bb6f-0547e8354fc2"
}
```

If we request a token with the wrong scope:

```sh
http POST $TOKEN_ENDPOINT \
    Authorization:$AUTH_HEADER \
    Content-Type:application/x-www-form-urlencoded \
    --raw "grant_type=client_credentials&scope=https://learninggolang.com/programming-jwtdebugger"
```

And then make a new call to the same endpoint but using this latest token:

```sh
http POST localhost:8080/v1/programming/uuid Authentication:eyJraWQiOiJrb1wvR2owY1...
```

The result should be similar to:

```json
HTTP/1.1 403 Forbidden
Content-Length: 23
Content-Type: application/json; charset=utf-8
Date: Fri, 07 Jan 2022 20:32:44 GMT

{
    "message": "Forbidden"
}
```

## Wrap up

Commit and push everything. Create a new tag.

```sh
git add .
git commit -m "feat: add authorization to the api"
git push
git tag -a v0.0.10 -m "v0.0.10"
git push origin v0.0.10
```