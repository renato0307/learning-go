# Add finance/currency-converter to the library

To convert an amount from a current (e.g. EUR) to another one (e.g USD) we
will use an external API.

There are several ones with free plans we can use for trial purposes.

We are going to use [FCS (Forex Crypto Stock)](https://fcsapi.com/) - the free
plan allows for 500 API calls per month.

The API to get conversion rates is very simple:

```
https://fcsapi.com/api-v3/forex/candle?symbol=EUR/USD&period=1h&access_key=XXX
```

This returns something like the following:

```json
HTTP/1.1 200 OK
Access-Control-Allow-Headers: X-Requested-With
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Origin: *
Cache-Control: max-age=31104000, private, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 223
Content-Type: application/json
Date: Mon, 27 Dec 2021 21:49:18 GMT
Expires: Wed, 26 Jan 2022 21:49:18 GMT
Keep-Alive: timeout=7, max=400
Server: Apache
Set-Cookie: c_reffer=direct; expires=Wed, 26-Jan-2022 21:49:18 GMT; Max-Age=2592000; path=/
Vary: Accept-Encoding

{
    "code": 200,
    "info": {
        "_t": "2021-12-27 21:49:18 UTC",
        "credit_count": 1,
        "server_time": "2021-12-27 21:49:18 UTC"
    },
    "msg": "Successfully",
    "response": [
        {
            "c": "1.13268",
            "ch": "-0.00013",
            "cp": "-0.01%",
            "h": "1.13281",
            "id": "1",
            "l": "1.13246",
            "o": "1.13281",
            "s": "EUR/USD",
            "t": "1640638800",
            "tm": "2021-12-27 21:00:00",
            "up": "2021-12-27 21:49:10"
        }
    ],
    "status": true
}
```

In the `response` field we can use the `c` value to do the conversion.

So, the first step is to create an account and get the API key available in the
dashboard.

## Changes in the library

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

Go to the Library folder.

We need to create `finance` folder and a few new files to support the
currency converter:

```sh
mkdir finance
touch finance/currconv.go
touch finance/currconv_test.go
touch finance/interface.go
```

The `finance/interface.go` is were we define the interface for the finance
a package:

```go
package finance

type Interface interface {
	ConvertCurrency(
		from string,
		to string,
		amount float64) (float64, error)
}

type FinanceFunctions struct {
	ApiUrl string
	ApiKey string
}
```

We can see here a difference from the other cases: the functions struct contains
fields. 

With this difference we are saying: to execute finance functions you need an 
API url and and API key. If several functions are to be executed, we can use
the same instance of the struct and we don't need to repeat those common
parameters on all the other functions.

After we define the interface, we can start the implementation, in the 
`finance/currconv.go` file.

We will need:
1. Structures to unmarshal the API response
1. A constructor function to create FinanceFunctions instances
1. The implementation

The complete code is presented below. Please read it, checking the comments.

```go
package finance

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"strconv"
	"strings"
)

const fcsapiUrl string = "https://fcsapi.com/api-v3/forex/candle?symbol=%s/%s&period=1h&access_key=%s"

// fasApiResponse represents the return from the Last Candle API from FSC.
// More details in the docs: https://fcsapi.com/document/forex-api#lastcandle
type fsaApiResponse struct {
	Code     int                     `json:"code"`
	Info     fsaApiResponseInfo      `json:"info"`
	Message  string                  `json:"msg"`
	Response []fsaApiResponseDetails `json:"response"`
}

type fsaApiResponseInfo struct {
	T           string `json:"_t"`
	CreditCount int    `json:"credit_count"`
	ServerTime  string `json:"server_time"`
}

type fsaApiResponseDetails struct {
	PriceClose           string `json:"c"`
	ChangeInOneDayCandle string `json:"ch"`
	ChangeInPercentage   string `json:"cp"`
	High                 string `json:"h"`
	ID                   string `json:"id"`
	Low                  string `json:"l"`
	Open                 string `json:"o"`
	Symbol               string `json:"s"`
	WhenUnix             string `json:"t"`
	WhenUtc              string `json:"tm"`
	WhenLastUpdateUtc    string `json:"up"`
}

// NewFinanceFunctions create a new FinanceFunctions instance. If the apiUrl is
// empty a default value will be set.
func NewFinanceFunctions(apiUrl, apiKey string) FinanceFunctions {
	ff := FinanceFunctions{
		ApiUrl: apiUrl,
		ApiKey: apiKey,
	}

	if ff.ApiUrl == "" {
		ff.ApiUrl = fcsapiUrl // sets the default value
	}

	return ff
}

// ConvertCurrency converts an amount from one currency into another using
// the https://fcsapi.com/ last candle API.
func (ff *FinanceFunctions) ConvertCurrency(
	from string,
	to string,
	amount float64) (float64, error) {

	response, err := ff.callLastCandleApi(from, to)
	if err != nil {
		return 0, err
	}

	priceClose, err := strconv.ParseFloat(response.Response[0].PriceClose, 64)
	if err != nil {
		err = fmt.Errorf("error parsing the conversion data: %s", err.Error())
		return 0, err
	}

	convertedAmount := priceClose * amount

	return convertedAmount, nil
}


// callLastCandleApi calls the Last Candle API, parsing the response into a
// struct.
func (ff *FinanceFunctions) callLastCandleApi(from, to string) (fsaApiResponse, error) {
	response := fsaApiResponse{}

	// calls the API and checks the result for errors
	url := ff.ApiUrl
	if strings.Count(ff.ApiUrl, "%s") > 0 { // when using httptest the URL contains no %s
		url = fmt.Sprintf(ff.ApiUrl, from, to, ff.ApiKey)
	}

	httpResponse, err := http.Get(url)
	if err != nil {
		err = fmt.Errorf("error getting the conversion data: %s", err.Error())
		return response, err
	}

	if httpResponse.StatusCode != http.StatusOK {
		err := fmt.Errorf("error getting the conversion data: %d", httpResponse.StatusCode)
		return response, err
	}

	defer httpResponse.Body.Close()

	// reads and response and converts JSON string to a struct
	body, err := ioutil.ReadAll(httpResponse.Body)
	if err != nil {
		err = fmt.Errorf("error reading the conversion data: %s", err.Error())
		return response, err
	}

	err = json.Unmarshal(body, &response)
	if err != nil {
		err = fmt.Errorf("error parsing the conversion data: %s", err.Error())
		return response, err
	}

	return response, nil
}
```
To complete the implementation we need to generate the mocks for the interface.

```sh
mockery -all -inpkg -case snake
```

## The unit tests for NewFinanceFunctions

The units tests for the NewFinanceFunctions function is going to be pretty
straightforward:

```go

func TestNewFinanceFunctions(t *testing.T) {
	// act
	ff := NewFinanceFunctions("DummyApiUrl", "DummyApiKey")

	// assert
	assert.Equal(t, "DummyApiUrl", ff.ApiUrl)
	assert.Equal(t, "DummyApiKey", ff.ApiKey)
}

func TestNewFinanceFunctionsWithEmptyApiUrl(t *testing.T) {
	// act
	ff := NewFinanceFunctions("", "DummyApiKey")

	// assert
	assert.NotEmpty(t, ff.ApiUrl)
	assert.Contains(t, ff.ApiUrl, "fcsapi.com")
}
```


## The unit tests for ConvertCurrency

The units tests are going to be a little more complex than the ones we
previously did.

As we are calling an external API, we need to mock it out to be able to
test all the necessary scenarios.

To helps doing that we'll use the 
[net/http/httptest](https://pkg.go.dev/net/http/httptest) package.

This package provides utilities for HTTP testing, namely to simulate a server.

The test for the happy flow is:

```go
func TestConvertCurrency(t *testing.T) {
	// arrange
	expected := `
		{
			"code": 200,
			"info": {
				"_t": "2021-12-27 21:49:18 UTC",
				"credit_count": 1,
				"server_time": "2021-12-27 21:49:18 UTC"
			},
			"msg": "Successfully",
			"response": [
				{
					"c": "1.13268",
					"ch": "-0.00013",
					"cp": "-0.01%",
					"h": "1.13281",
					"id": "1",
					"l": "1.13246",
					"o": "1.13281",
					"s": "EUR/USD",
					"t": "1640638800",
					"tm": "2021-12-27 21:00:00",
					"up": "2021-12-27 21:49:10"
				}
			],
			"status": true
		}
	`
	svr := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, expected)
	}))
	defer svr.Close()

	ff := NewFinanceFunctions(svr.URL, "DummyApiKey")

	// act
	result, err := ff.ConvertCurrency("EUR", "USD", 10)

	// assert
	assert.Nil(t, err)
	assert.Equal(t, 11.3268, math.Round(result*10000)/10000)
}
```

The `httptest.NewServer` will start a local server which will return a
predefined response.

The output of this function is a `httptest.Server` struct containing an URL
we can use to call the `NewFinanceFunctions` constructor. 

So when we call the `ConvertCurrency` function, the URL for the test server will
be used instead of a real one.

The rest of the test is just simple asserts over the result.

As Go does not have a standard library to round numbers with a certain number of
decimal places, we use `math.Round(result*10000)/10000)` to achieve the same 
result.

🏋️‍♀️ __CHALLENGE__: implement the tests for error cases by yourself before
proceeding.

The complete code for the tests is:

```go
package finance

import (
	"fmt"
	"math"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestNewFinanceFunctions(t *testing.T) {
	// act
	ff := NewFinanceFunctions("DummyApiUrl", "DummyApiKey")

	// assert
	assert.Equal(t, "DummyApiUrl", ff.ApiUrl)
	assert.Equal(t, "DummyApiKey", ff.ApiKey)
}

func TestNewFinanceFunctionsWithEmptyApiUrl(t *testing.T) {
	// act
	ff := NewFinanceFunctions("", "DummyApiKey")

	// assert
	assert.NotEmpty(t, ff.ApiUrl)
	assert.Contains(t, ff.ApiUrl, "fcsapi.com")
}

func TestConvertCurrency(t *testing.T) {
	// arrange
	expected := `
		{
			"code": 200,
			"info": {
				"_t": "2021-12-27 21:49:18 UTC",
				"credit_count": 1,
				"server_time": "2021-12-27 21:49:18 UTC"
			},
			"msg": "Successfully",
			"response": [
				{
					"c": "1.13268",
					"ch": "-0.00013",
					"cp": "-0.01%",
					"h": "1.13281",
					"id": "1",
					"l": "1.13246",
					"o": "1.13281",
					"s": "EUR/USD",
					"t": "1640638800",
					"tm": "2021-12-27 21:00:00",
					"up": "2021-12-27 21:49:10"
				}
			],
			"status": true
		}
	`
	svr := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, expected)
	}))
	defer svr.Close()

	ff := NewFinanceFunctions(svr.URL, "DummyApiKey")

	// act
	result, err := ff.ConvertCurrency("EUR", "USD", 10)

	// assert
	assert.Nil(t, err)
	assert.Equal(t, 11.3268, math.Round(result*10000)/10000)
}

func TestConvertCurrencyWithApiError(t *testing.T) {
	// arrange
	svr := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(400)
	}))
	defer svr.Close()

	ff := NewFinanceFunctions(svr.URL, "DummyApiKey")

	// act
	result, err := ff.ConvertCurrency("EUR", "USD", 10)

	// assert
	assert.NotNil(t, err)
	assert.Equal(t, 0.0, result)
}

func TestConvertCurrencyWithInvalidJsonBody(t *testing.T) {
	// arrange
	expected := "invalid json"

	svr := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, expected)
	}))
	defer svr.Close()

	ff := NewFinanceFunctions(svr.URL, "DummyApiKey")

	// act
	result, err := ff.ConvertCurrency("EUR", "USD", 10)

	// assert
	assert.NotNil(t, err)
	assert.Equal(t, 0.0, result)
}
```

### Wrapping

Commit and push everything. Create a new tag.

```sh
git add .
git commit -m "feat: add finance/currconv"
git push
git tag -a v0.0.3 -m "v0.0.3"
git push origin v0.0.3
```