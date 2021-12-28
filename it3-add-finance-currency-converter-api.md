# Add finance/currency-converter to the API

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

We are going to add support to execute a currency conversion when a `GET`
request is sent to `/finance/currconv`.

Go to the API folder.

The first change is to update the Library version and run `go mod tidy`:

```go.mod
module github.com/renato0307/learning-go-api

go 1.17

require github.com/gin-gonic/gin v1.7.7

require (
	github.com/renato0307/learning-go-lib v0.0.4 // change
	github.com/stretchr/testify v1.7.0
)

// ...continues
```

We need to create `finance` folder and a few new files to support the
currency converter:

```sh
mkdir finance
touch finance/finance.go
touch finance/finance_test.go
```

The first change is to add a `SetRouterGroup` in the `finance/finance.go` file
so we can define routes for the `finance` category:

```go
package finance

import (
	"github.com/gin-gonic/gin"
	"github.com/renato0307/learning-go-lib/finance"
)

// SetRouterGroup defines all the routes for the finance functions
func SetRouterGroup(f finance.Interface, base *gin.RouterGroup) *gin.RouterGroup {
	financeGroup := base.Group("/finance")
	{
		financeGroup.GET("/currconv", getCurrConv(f))
		// Add here more functions in the finance category
	}

	return financeGroup
}

func getCurrConv(f finance.Interface) gin.HandlerFunc {
	return func(c *gin.Context) {
		// TODO
	}
}
```

For this to work we also need call the `SetRouterGroup` in `main.go`:

```go
package main

import (
	"fmt"
	"os"

	"github.com/gin-gonic/gin"
	"github.com/renato0307/learning-go-api/finance" // new
	"github.com/renato0307/learning-go-api/programming"
	financelib "github.com/renato0307/learning-go-lib/finance" // new
	programminglib "github.com/renato0307/learning-go-lib/programming"
)

func main() {
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "Hello, welcome to the learning-go-api",
		})
	})

	base := r.Group("/v1")

	p := programminglib.ProgrammingFunctions{}
	programming.SetRouterGroup(&p, base)

	useDefaultUrl := ""                                        // new
	apiKey := getRequiredEnv("CURRCONV_API_KEY")               // new
	f := financelib.NewFinanceFunctions(useDefaultUrl, apiKey) // new
	finance.SetRouterGroup(&f, base)                           // new

	r.Run()
}

func getRequiredEnv(key string) string { // new
	value, exists := os.LookupEnv(key)

	if !exists {
		panic(fmt.Sprintf("error: %s environment variable was not defined", key))
	}

	return value
}

```

For us to be able to call the finance functions we need to have an API key. This
will be a configuration of the API, which will be passed by the use of an
environment variable. Some of the extra code (like the `getRequiredEnv`
function) serves that purpose.

With this change, before starting the API we need to define a new environment
variable.

For example:

```sh
CURRCONV_API_KEY=api_key_value_here go run main.go
```

----
🕵️‍♀️ __GO-EXTRA: Panic__

We have used the `panic` built-in function to raise an error and interrupt
the execution of the code if a required environment variable is missing.

For more information about panic and recover check
[this blog post](https://go.dev/blog/defer-panic-and-recover).

----

Now let's implement the `getCurrConv` function in the `finance/finance.go` file:

```go
// getCurrConv handles the currency conversion request.
//
// The request requires the from, to and amount parameters in the query string.
// It returns HTTP 200 on success.
// Returns HTTP 400 if there is a missing parameter.
// Returns HTTP 500 if there is another error.
func getCurrConv(f finance.Interface) gin.HandlerFunc {
	return func(c *gin.Context) {
		from := c.Query("from")
		to := c.Query("to")
		amount := c.Query("amount")

		if from == "" {
			c.JSON(http.StatusBadRequest, "error: 'from' parameter is required")
			return
		}

		if to == "" {
			c.JSON(http.StatusBadRequest, "error: 'to' parameter is required")
			return
		}

		if amount == "" {
			c.JSON(http.StatusBadRequest, "error: 'amount' parameter is required")
			return
		}

		amountFloat, err := strconv.ParseFloat(amount, 64)
		if err != nil {
			c.JSON(http.StatusBadRequest, "error: 'amount' is not a valid number")
			return
		}

		convertAmount, err := f.ConvertCurrency(from, to, amountFloat)
		if err != nil {
			err = fmt.Errorf("error converting the currency: %s", err.Error())
			c.JSON(http.StatusInternalServerError, err.Error())
		}

		output := currConvOutput{
			From:            from,
			To:              to,
			Amount:          amountFloat,
			ConvertedAmount: convertAmount,
		}

		c.JSON(http.StatusOK, output)
	}
}
```

The code is pretty explanatory so let's proceed to the tests.

## The unit tests for main.go

As we added some logic to the `main.go`file, let's add some tests:

```sh
touch main_test.go
```

The contents of the test are:

```go
package main

import (
	"os"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestGetRequiredEnv(t *testing.T) {
	// arrange
	varName := "ENV_VAR_NAME"
	varValue := "ENV_VAR_VALUE"

	os.Setenv(varName, varValue)

	// act
	value := getRequiredEnv(varName)

	// assert
	assert.Equal(t, varValue, value)
}

func TestGetRequiredEnvWithMissingEnvironment(t *testing.T) {
	// act & assert
	assert.Panics(t, func() {
		getRequiredEnv("ENV_VAR_NAME")
	})
}
```

The I want to highlight the usage of the `assert.Panics` function to check if
the `getRequiredEnv` panics when an environment variable is missing.

## The unit tests for finance/finance.go

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

The test for the success scenario is:

```go
func TestGetCurrConv(t *testing.T) {
	// arrange
	from := "EUR"
	to := "USD"
	amount := 10.0
	amountConverted := 11.0

	mockInterface := financelib.MockInterface{}
	mockCall := mockInterface.On("ConvertCurrency", from, to, amount)
	mockCall.Return(amountConverted, nil)

	r := setupGin(&mockInterface)
	w := httptest.NewRecorder()

	url := fmt.Sprintf("/v1/finance/currconv?from=%s&to=%s&amount=%f", from, to, amount)
	req, _ := http.NewRequest("GET", url, nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusOK)

	output := getCurrConvOutput{}
	err := json.Unmarshal(w.Body.Bytes(), &output)

	assert.Nil(t, err)
	assert.Equal(t, from, output.From)
	assert.Equal(t, to, output.To)
	assert.Equal(t, amount, output.Amount)
	assert.Equal(t, amountConverted, output.ConvertedAmount)

	mockInterface.AssertExpectations(t)
}
```

The full tests, including the error scenarios are:

```go
package finance

import (
	"encoding/json"
	"errors"
	"fmt"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/gin-gonic/gin"
	financelib "github.com/renato0307/learning-go-lib/finance"
	"github.com/stretchr/testify/assert"
)

func setupGin(mockInterface *financelib.MockInterface) *gin.Engine {
	r := gin.Default()
	v1 := r.Group("/v1")
	SetRouterGroup(mockInterface, v1)

	return r
}

func TestGetCurrConv(t *testing.T) {
	// arrange
	from := "EUR"
	to := "USD"
	amount := 10.0
	amountConverted := 11.0

	mockInterface := financelib.MockInterface{}
	mockCall := mockInterface.On("ConvertCurrency", from, to, amount)
	mockCall.Return(amountConverted, nil)

	r := setupGin(&mockInterface)
	w := httptest.NewRecorder()

	url := fmt.Sprintf("/v1/finance/currconv?from=%s&to=%s&amount=%f", from, to, amount)
	req, _ := http.NewRequest("GET", url, nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusOK)

	output := getCurrConvOutput{}
	err := json.Unmarshal(w.Body.Bytes(), &output)

	assert.Nil(t, err)
	assert.Equal(t, from, output.From)
	assert.Equal(t, to, output.To)
	assert.Equal(t, amount, output.Amount)
	assert.Equal(t, amountConverted, output.ConvertedAmount)

	mockInterface.AssertExpectations(t)
}

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
}

func TestGetCurrConvWithMissingTo(t *testing.T) {
	// arrange
	from := "EUR"
	amount := 10.0

	mockInterface := financelib.MockInterface{}
	r := setupGin(&mockInterface)
	w := httptest.NewRecorder()

	url := fmt.Sprintf("/v1/finance/currconv?from=%s&amount=%f", from, amount)
	req, _ := http.NewRequest("GET", url, nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusBadRequest)
}

func TestGetCurrConvWithMissingAmount(t *testing.T) {
	// arrange
	from := "EUR"
	to := "USD"

	mockInterface := financelib.MockInterface{}
	r := setupGin(&mockInterface)
	w := httptest.NewRecorder()

	url := fmt.Sprintf("/v1/finance/currconv?from=%s&to=%s", from, to)
	req, _ := http.NewRequest("GET", url, nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusBadRequest)
}

func TestGetCurrConvWithInvalidAmount(t *testing.T) {
	// arrange
	from := "EUR"
	to := "USD"
	amount := "invalid"

	mockInterface := financelib.MockInterface{}
	r := setupGin(&mockInterface)
	w := httptest.NewRecorder()

	url := fmt.Sprintf("/v1/finance/currconv?from=%s&to=%s&amount=%s", from, to, amount)
	req, _ := http.NewRequest("GET", url, nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusBadRequest)
}

func TestGetCurrConvWithLibraryError(t *testing.T) {
	// arrange
	from := "EUR"
	to := "USD"
	amount := 10.0

	mockInterface := financelib.MockInterface{}
	mockCall := mockInterface.On("ConvertCurrency", from, to, amount)
	mockCall.Return(0.0, errors.New("fake error"))

	r := setupGin(&mockInterface)
	w := httptest.NewRecorder()

	url := fmt.Sprintf("/v1/finance/currconv?from=%s&to=%s&amount=%f", from, to, amount)
	req, _ := http.NewRequest("GET", url, nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusInternalServerError)
	mockInterface.AssertExpectations(t)
}

```

## Manual tests

To run manual tests first start the API using the correct API key created
before:

```sh
CURRCONV_API_KEY=<PUT-HERE-THE-API-KEY> go run main.go
```

Next, in a new terminal window, use `httpie` to do a couple of calls.

The success call is:

```sh
http "localhost:8080/v1/finance/currconv?from=EUR&to=USD&amount=200"
```

The result should be similar to:

```json
HTTP/1.1 200 OK
Content-Length: 76
Content-Type: application/json; charset=utf-8
Date: Tue, 28 Dec 2021 06:47:24 GMT

{
    "amount": 200,
    "converted_amount": 226.42600000000002,
    "from": "EUR",
    "to": "USD"
}
```