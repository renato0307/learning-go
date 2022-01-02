# Improve API error handling

All success responses from the API return a JSON object. When we have errors,
a plain string is returned instead. 

In this section we are going to improve this, by returning a JSON object when
errors occur.

We will need to:

1. Create a structure to represent an API error
1. Use the structure to return an JSON object when an error occurs
1. Validate the return in the API tests

## Creating the structure

Go to the API directory.

The error structure needs to be used in all packages.

So it needs to go to a separated package, named `apierror`.

Let's create the `apierror` directory and add the structure definition.

```sh
mkdir apierror
touch apierror/apierror.go
touch apierror/apierror_test.go
```

The contents of the `apierror.go` are:

```go
package apierror

import (
	"encoding/json"
	"testing"

	"github.com/stretchr/testify/assert"
)

type ApiError struct {
	Message string `json:"message"`
}

func New(message string) ApiError {
	return ApiError{Message: message}
}
```

Let's add a test for the `New` constructor:

```go
package apierror

import (
	"encoding/json"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestNewApiError(t *testing.T) {

	errorMessage := "this is a fake error message"
	err := New(errorMessage)

	assert.Equal(t, errorMessage, err.Message)
}
```

## Use the API error structure

Let's start by the `finance/currconv.go` file.

The errors are handled like the following example:

```go
        // ...
        if from == "" {
            c.JSON(http.StatusBadRequest, "error: 'from' parameter is required")
            return
        }
        // ...
```

We need to return an `ApiError` instead of a plain string:

```go
        // ...
		if from == "" {
			msg := "error: 'from' parameter is required"
			c.JSON(http.StatusBadRequest, apierror.New(msg))
			return
		}
        // ...
```

🏋️‍♀️ __CHALLENGE__: Do this change for all errors in all packages.

## Change the tests

In the error test cases we need to check if the return is valid.

The code to verify if the error code is valid is:

```go
apiError := ApiError{}
err := json.Unmarshal(jsonData, &apiError)

assert.Nil(t, err)
assert.NotEmpty(t, apiError.Message)

```

We can add this as a function in the `apierror` package so we can keep our tests
free of duplicated code. The contents of the `apierror.go` file are:

```go
package apierror

import (
	"encoding/json"
	"testing"

	"github.com/stretchr/testify/assert"
)

type ApiError struct {
	Message string `json:"message"`
}

func New(message string) ApiError {
	return ApiError{Message: message}
}

func AssertIsValid(t *testing.T, jsonData []byte) { // ne
	apiError := ApiError{}
	err := json.Unmarshal(jsonData, &apiError)

	assert.Nil(t, err)
	assert.NotEmpty(t, apiError.Message)

}
```

If the `currconv_test.go` we can now use the `AssertIsValid` function to check
the return.

For example:

```go
func TestGetCurrConvWithMissingFrom(t *testing.T) {
	// arrange
	to := "USD"
	amount := 10.0

	mockInterface := financelib.MockInterface{}
	r := setupGin(&mockInterface)
	w := httptest.NewRecorder()

	url := fmt.Sprintf("/v1/finance/currconv?to=%s&amount=%f", to, amount)
	req, _ := http.NewRequest("GET", url, nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusBadRequest)
	apierror.AssertIsValid(t, w.Body.Bytes()) // asserting the error
}
```

🏋️‍♀️ __CHALLENGE__: Do this change for the tests in all packages.

In the end, ensure all tests are green.

## Manual testing

To see the output, let's do an invalid request using `httpie`.

Start the API:

```sh
go run main.go
```

And make the request:

```sh
http localhost:8080/v1/finance/currconv
```

The result should be similar to:

```json
HTTP/1.1 400 Bad Request
Content-Length: 49
Content-Type: application/json; charset=utf-8
Date: Sun, 02 Jan 2022 11:47:24 GMT

{
    "message": "error: 'from' parameter is required"
}
```

## Wrap up

Commit and push everything. Create a new tag.

```sh
git add .
git commit -m "refactor: return json on error cases"
git push
git tag -a v0.0.6 -m "v0.0.6"
git push origin v0.0.6
```

After some minutes we should have the new version installed in our k8s cluster.