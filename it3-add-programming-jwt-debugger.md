# Add programming/jwtdebugger

The goal of this feature is to allow to inspect JSON Web Tokens.

If you don't know what a JSON Web Token is, check
[this introduction](https://jwt.io/introduction).

From the [jwt.io](jwt.io):
> JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact
> and self-contained way for securely transmitting information between parties 
> as a JSON object. This information can be verified and trusted because it is
> digitally signed.

In its compact form, JSON Web Tokens consist of three parts separated by dots
(.), which are:

* Header
* Payload
* Signature

A JWT typically looks like the following

`xxxxx.yyyyy.zzzzz`

It is common when using JWT to have the need to inspect it, for troubleshooting
purposes.

For example, we might need to check payload to understand the permissions
associated with the token or its validity, in case authentication issues.

From a token like

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```
We can extract the header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

and the payload

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```
The goal of this feature is to get the header and payload from a compacted JWT.

__CHALLENGE__: try to implement this by yourself before proceeding. 

## Changes in the Library

Go to the Library folder.

We need to create two new files to support the JWT debugger:

```sh
echo "package programming" > programming/jwtdebugger.go
echo "package programming" > programming/jwtdebugger_test.go
```

We are going to use the [github.com/golang-jwt/jwt]("github.com/golang-jwt/jwt")
to parse the tokens.

The contents of the `programming/jwtdebugger.go` are:

```go
package programming

import (
	"encoding/json"
	"fmt"

	"github.com/golang-jwt/jwt"
)

// DebugJWT parses a JWT and returns the header and payload contents
// WARNING: this function does not validate the token, only inspects the content
func (pf *ProgrammingFunctions) DebugJWT(tokenString string) (string, string, error) {

	parser := jwt.Parser{}
	token, _, err := parser.ParseUnverified(tokenString, jwt.MapClaims{})
	if err != nil {
		return "", "", fmt.Errorf("error parsing token: %s", err.Error())
	}

	header, _ := json.Marshal(token.Header)
	payload, _ := json.Marshal(token.Claims)

	return string(header), string(payload), nil
}
```

The contents of the `programming/jwtdebugger_test.go` are:

```go
package programming

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestDebugJWT(t *testing.T) {
	// arrange
	var pf ProgrammingFunctions = ProgrammingFunctions{}
	tokenString := "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"

	// act
	header, payload, err := pf.DebugJWT(tokenString)

	// assert
	expectedHeader := "{\"alg\":\"HS256\",\"typ\":\"JWT\"}"
	expectedPayload := "{\"iat\":1516239022,\"name\":\"John Doe\",\"sub\":\"1234567890\"}"

	assert.Nil(t, err)
	assert.Equal(t, expectedHeader, header)
	assert.Equal(t, expectedPayload, payload)
}

func TestDebugJWTWithInvalidToken(t *testing.T) {
	// arrange
	var pf ProgrammingFunctions = ProgrammingFunctions{}
	tokenString := "xxxxx.yyyyy.zzzzz"

	// act
	header, payload, err := pf.DebugJWT(tokenString)

	// assert
	assert.NotNil(t, err)
	assert.Contains(t, err.Error(), "error parsing token")
	assert.Empty(t, header)
	assert.Empty(t, payload)
}
```

The tests should all pass:

```sh
go test ./... -v
```

The test result is:

```
=== RUN   TestDebugJWT
--- PASS: TestDebugJWT (0.00s)
=== RUN   TestDebugJWTWithInvalidToken
--- PASS: TestDebugJWTWithInvalidToken (0.00s)
=== RUN   TestNewUuidWithHyphen
--- PASS: TestNewUuidWithHyphen (0.00s)
=== RUN   TestNewUuidWithoutHyphen
--- PASS: TestNewUuidWithoutHyphen (0.00s)
PASS
ok      github.com/renato0307/learning-go-lib/programming       0.002s
```

If you recall from
[unit tests in the API using mocks](it1-api-unit-tests-with-mocks.md) we need
to add the `DebugJWT` function definition to the `interface.go` file:

```go
package programming

type Interface interface {
	NewUuid(withoutHyphen bool) string
	DebugJWT(tokenString string) (string, string, error) // new
}

type ProgrammingFunctions struct {
}
```

We also need to generate the mocks:

```sh
mockery -all -inpkg -case snake
```

After this is done, commit and push the changes to GitHub.

To finish, create a new tag.

```sh
git add .
git commit -m "feat: add programming/jwtdebugger"
git push
git tag -a v0.0.3 -m "v0.0.3"
git push origin v0.0.3
```

## Changes in the API