# CLI building blocks - authentication

In each command we need to get authentication tokens.

This implies:

1. Getting the configurations
1. Calling AWS Cognito OAuth2 token endpoints
1. Parse the result into a structure

This package will be named `auth` and placed in `internal/auth`.

```sh
mkdir -p internal/auth
touch internal/auth/auth.go
touch internal/auth/auth_test.go
```

The logic to be implemented is described in the
[AWS documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html).

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

The contents of the `auth.go` file is:

```go
package auth

import (
	"encoding/base64"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"

	"github.com/renato0307/learning-go-cli/internal/config"
)

// AccessToken represents an OAuth2 access token obtained using the client
// credentials flow
type AccessToken struct {
	AccessToken string `json:"access_token"`
	ExpiresIn   int    `json:"expires_in"`
	TokenType   string `json:"token_type"`
}

// NewAccessToken fetches a new access token from the OAuth2 server
func NewAccessToken() (AccessToken, error) {
	accessToken := AccessToken{}

	// get configurations
	clientId := config.GetString(config.ClientIdFlag)
	clientSecret := config.GetString(config.ClientSecretFlag)
	tokenEndpoint := config.GetString(config.TokenEndpointFlag)

	// prepare request body
	bodyContent := fmt.Sprintf(
		"grant_type=client_credentials&client_id=%s&scope=",
		clientId)
	body := strings.NewReader(bodyContent)

	// create base request
	request, err := http.NewRequest("POST", tokenEndpoint, body)
	if err != nil {
		return accessToken, err
	}

	// set the headers
	clientIdAndSecret := fmt.Sprintf("%s:%s", clientId, clientSecret)
	credentials := base64.StdEncoding.EncodeToString([]byte(clientIdAndSecret))
	authHeader := fmt.Sprintf("Basic %s", credentials)
	request.Header = map[string][]string{
		"Authorization": {authHeader},
		"Content-Type":  {"application/x-www-form-urlencoded"},
	}

	// execute the request
	response, err := http.DefaultClient.Do(request)
	if err != nil {
		return accessToken, err
	}

	// read and unmarshal the body
	responseContent, err := ioutil.ReadAll(response.Body)
	if err != nil {
		return accessToken, err
	}
	defer response.Body.Close()
	if response.StatusCode != 200 {
		return accessToken, fmt.Errorf("error getting token: %s", responseContent)
	}
	err = json.Unmarshal(responseContent, &accessToken)

	return accessToken, err
}
```

## Unit tests

For the unit tests we will use the `httptest` package to simulate the token
endpoint.

We will also define all test cases in a list and run them all using a loop like
we did before:

```go
package auth

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/renato0307/learning-go-cli/internal/config"
	"github.com/stretchr/testify/assert"
)

func TestNewAccessToken(t *testing.T) {

	testCases := []struct {
		Token      AccessToken
		Raw        string
		StatusCode int
		Purpose    string
		ErrorNil   bool
	}{
		{
			Token: AccessToken{
				AccessToken: "token",
				ExpiresIn:   1000,
				TokenType:   "Bearer",
			},
			StatusCode: 200,
			Purpose:    "success case",
			ErrorNil:   true,
		},
		{
			Token:      AccessToken{},
			StatusCode: 500,
			Purpose:    "get token failure case",
			ErrorNil:   false,
		},
		{
			Raw:        "this_is_invalid_json",
			StatusCode: 200,
			Purpose:    "invalid token",
			ErrorNil:   false,
		},
	}

	for _, tc := range testCases {
		// arrange
		srv := httptest.NewServer(
			http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
				// to test the code sends an authorization token
				if r.Header.Get("Authorization") == "" {
					w.WriteHeader(http.StatusBadRequest)
					w.Write([]byte("Unauthorized"))
					return
				}

				// writes the response for other cases
				// we use the Raw field to form malformed responses
				w.WriteHeader(tc.StatusCode)
				if tc.Raw != "" {
					w.Write([]byte(tc.Raw))
				} else {
					body, _ := json.Marshal(tc.Token)
					w.Write(body)
				}
			}))
		defer srv.Close()

		config.Set(config.TokenEndpointFlag, srv.URL)

		// act
		token, err := NewAccessToken()

		// assert
		if tc.ErrorNil {
			assert.NoError(t, err, "error found for "+tc.Purpose)
		} else {
			assert.Error(t, err, "error not found for "+tc.Purpose)
		}
		assert.Equal(t, tc.Token, token, "invalid token for "+tc.Purpose)
	}
}
```

# Next
 
The next section is
[CLI building blocks: IO streams & testing helpers](it7-cli-building-blocks-iostreams.md).