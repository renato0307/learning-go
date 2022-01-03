# Add structured logs to the API

One of the problems of the current API implementation is the lack of logs,
besides the ones already added out-of-the-box by Gin.

Logs are one of the most valuable assets a developer can have for
troubleshooting purposes.

As we are adding logs from scratch, we are going to use `structured logging`,
which brings more structure and details to our logs.

The standard logging library does not support structured logging by default,
so we are going to use one of the external loggers supporting it.

Our pick is [rs/zerolog](https://github.com/rs/zerolog).

## Basic logging

Let's start by a simple example in the `finance/finance.go`:

```go
package finance

import (
	"github.com/gin-gonic/gin"
	"github.com/renato0307/learning-go-lib/finance"

	"github.com/rs/zerolog/log" // new
)

// SetRouterGroup defines all the routes for the finance functions
func SetRouterGroup(f finance.Interface, base *gin.RouterGroup) *gin.RouterGroup {
	
    log.Debug().Msg("setting router group for: finance") // new

	financeGroup := base.Group("/finance")
	{
		financeGroup.GET("/currconv", getCurrConv(f))
		// Add here more functions in the finance category
	}

	return financeGroup
}
```

We have two simple changes:

1. Importing the log library
1. Send a `Debug` message when setting the router group for the finance category

To see the full potential of structured login we need another example.

In the `finance/currconv.go` file:

```go
package finance

import (
	"fmt"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/renato0307/learning-go-api/apierror"
	"github.com/renato0307/learning-go-lib/finance"
	"github.com/rs/zerolog/log" // new
)

type getCurrConvOutput struct {
	From            string  `json:"from"`
	To              string  `json:"to"`
	Amount          float64 `json:"amount"`
	ConvertedAmount float64 `json:"converted_amount"`
}

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

		log.Debug().
			Str("from", from).
			Str("to", to).
			Str("amount", amount).
			Msg("running currency converter") // new

		if from == "" {
			msg := "error: 'from' parameter is required"
			c.JSON(http.StatusBadRequest, apierror.New(msg))
			return
		}
// (...)
```

In the case we can see:

1. Importing the log library
1. Send a `Debug` message with three additional `string` fields

Let's see the result of these changes.

Start the Gin server:

```sh
CURRCONV_API_KEY=apikeyvalue GIN_MODE=release go run main.go
```

Then use `httpie` do send a currency convert request:

```sh
http "localhost:8080/v1/finance/currconv?from=EUR&to=USD"
```

In the terminal running Gin we can see the following log statements:

```
{"level":"debug","time":"2022-01-02T21:08:58Z","message":"setting router group for: finance"}
{"level":"debug","from":"EUR","to":"USD","amount":"","time":"2022-01-02T21:11:40Z","message":"running currency converter"}
```

I would like to highlight:

1. The log entry generate JSON instead of plain text
1. The second log entry shows the extra fields added

🏋️‍♀️ __CHALLENGE__: using the examples go over the existing code and add log
statements.

## Wrap up

Commit and push everything. Create a new tag.

```sh
git add .
git commit -m "refactor: add log statements"
git push
git tag -a v0.0.7 -m "v0.0.7"
git push origin v0.0.7
```