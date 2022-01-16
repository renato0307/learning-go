# Make Gin use structured logs

After adding structured logs to the API, if we start Gin and make a request, we
will see mixed log formats. Our new log entries have a JSON format but the
default log entries from Gin are still plain text:

```terminal
{"level":"debug","time":"2022-01-03T07:20:28Z","message":"setting router group for: programming"}
{"level":"debug","time":"2022-01-03T07:20:28Z","message":"setting router group for: finance"}
{"level":"debug","from":"","to":"","amount":"","time":"2022-01-03T07:20:33Z","message":"running currency converter"}
[GIN] 2022/01/03 - 07:20:33 | 400 |       44.03µs |       127.0.0.1 | GET      "/v1/finance/currconv"
```

To change this we need to do two things:

1. Create a new Gin middleware to write logs using our structured logger
1. Configure Gin to use the new logging middleware instead of the default one

## Creating a logging middleware

Let's create a `middleware` folder to keep the logger:

```sh
mkdir middleware
touch middleware/logger.go
```

The contents of the `logger.go` file are:

```go
package middleware

import (
	"time"

	"github.com/gin-gonic/gin"
	"github.com/rs/zerolog"
	"github.com/rs/zerolog/log"
)


// DefaultStructuredLogger logs a gin HTTP request in JSON format. Uses the
// default logger from rs/zerolog.
func DefaultStructuredLogger() gin.HandlerFunc {
	return StructuredLogger(&log.Logger)
}

// StructuredLogger logs a gin HTTP request in JSON format. Allows to set the
// logger for testing purposes.
func StructuredLogger(logger *zerolog.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {

		start := time.Now() // Start timer
		path := c.Request.URL.Path
		raw := c.Request.URL.RawQuery

		// Process request
		c.Next()

		// Fill the params
		param := gin.LogFormatterParams{}

		param.TimeStamp = time.Now() // Stop timer
		param.Latency = param.TimeStamp.Sub(start)
		if param.Latency > time.Minute {
			param.Latency = param.Latency.Truncate(time.Second)
		}

		param.ClientIP = c.ClientIP()
		param.Method = c.Request.Method
		param.StatusCode = c.Writer.Status()
		param.ErrorMessage = c.Errors.ByType(gin.ErrorTypePrivate).String()
		param.BodySize = c.Writer.Size()
		if raw != "" {
			path = path + "?" + raw
		}
		param.Path = path

		// Log using the params
		var logEvent *zerolog.Event
		if c.Writer.Status() >= 500 {
			logEvent = logger.Error()
		} else {
			logEvent = logger.Info()
		}

		logEvent.Str("client_id", param.ClientIP).
			Str("method", param.Method).
			Int("status_code", param.StatusCode).
			Int("body_size", param.BodySize).
			Str("path", param.Path).
			Str("latency", param.Latency.String()).
			Msg(param.ErrorMessage)
	}
}
```

This middleware replicates part of the logic of the
[default logger](https://github.com/gin-gonic/gin/blob/94153d1e19e20f56f520b31007e667ab6a5caeab/logger.go#L233),
picking the information from the request context, calculates the latency and
writes the log entry using `rs/zerolog`.

## Configure Gin

To configure Gin to use the new logger middleware we need to modify `main.go`.

Instead of using `r := gin.Default()` we will need a new blank Engine instance
without any middleware attached.

If you check the `gin.Default()` code we can see two middlewares in use:
* `Logger`
* `Recovery`

```go
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```

So, to do a similar setup we need to use our new logger middleware but still add
the `Recovery` one added by the default engine.

The complete code is available below:

```go
package main

import (
	"fmt"
	"os"

	"github.com/gin-gonic/gin"
	"github.com/renato0307/learning-go-api/finance"
	"github.com/renato0307/learning-go-api/middleware"
	"github.com/renato0307/learning-go-api/programming"
	financelib "github.com/renato0307/learning-go-lib/finance"
	programminglib "github.com/renato0307/learning-go-lib/programming"
)

func main() {
	// Initialize Gin
	gin.SetMode(gin.ReleaseMode)
	r := gin.New()                              // empty engine
	r.Use(middleware.DefaultStructuredLogger()) // adds our new middleware
	r.Use(gin.Recovery())                       // adds the default recovery middleware

	// Default route
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "Hello, welcome to the learning-go-api",
		})
	})

	// Utility functions routes
	base := r.Group("/v1")

	p := programminglib.ProgrammingFunctions{}
	programming.SetRouterGroup(&p, base)

	useDefaultUrl := ""
	apiKey := getRequiredEnv("CURRCONV_API_KEY")
	f := financelib.NewFinanceFunctions(useDefaultUrl, apiKey)
	finance.SetRouterGroup(&f, base)

	// Start serving request
	r.Run()
}

func getRequiredEnv(key string) string {
	value, exists := os.LookupEnv(key)

	if !exists {
		panic(fmt.Sprintf("error: %s environment variable was not defined", key))
	}

	return value
}
```

## Unit testing

The contents for the `middleware/logger_test.go` file are:

```go
package middleware

import (
	"bytes"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/gin-gonic/gin"
	"github.com/rs/zerolog"
	"github.com/stretchr/testify/assert"
)

func PerformRequest(r http.Handler, method, path string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, path, nil)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)
	return w
}

func TestStructuredLogger(t *testing.T) {
	// arrange - create a new logger writing to a buffer
	buffer := new(bytes.Buffer)
	var memLogger = zerolog.New(buffer).With().Timestamp().Logger()

	// arrange - init Gin to use the structured logger middleware
	r := gin.New()
	r.Use(StructuredLogger(&memLogger))
	r.Use(gin.Recovery())

	// arrange - set the routes
	r.GET("/example", func(c *gin.Context) {})
	r.GET("/force500", func(c *gin.Context) { panic("forced panic") })

	// act & assert
	PerformRequest(r, "GET", "/example?a=100")
	assert.Contains(t, buffer.String(), "200")
	assert.Contains(t, buffer.String(), "GET")
	assert.Contains(t, buffer.String(), "/example")
	assert.Contains(t, buffer.String(), "a=100")

	buffer.Reset()
	PerformRequest(r, "GET", "/notfound")
	assert.Contains(t, buffer.String(), "404")
	assert.Contains(t, buffer.String(), "GET")
	assert.Contains(t, buffer.String(), "/notfound")

	buffer.Reset()
	PerformRequest(r, "GET", "/force500")
	assert.Contains(t, buffer.String(), "500")
	assert.Contains(t, buffer.String(), "GET")
	assert.Contains(t, buffer.String(), "/force500")
	assert.Contains(t, buffer.String(), "error")
}
```

## Manual testing

First start Gin:

```sh
CURRCONV_API_KEY=apikeyvalue go run main.go
```

Then use `httpie` do send a currency convert request:

```sh
http "localhost:8080/v1/finance/currconv?from=EUR&to=USD"
```

In the terminal running Gin we can see the following log statements, all using
JSON:

```
{"level":"debug","time":"2022-01-03T07:41:42Z","message":"setting router group for: programming"}
{"level":"debug","time":"2022-01-03T07:41:42Z","message":"setting router group for: finance"}
{"level":"debug","from":"EUR","to":"USD","amount":"","time":"2022-01-03T07:43:11Z","message":"running currency converter"}
{"level":"info","client_id":"127.0.0.1","method":"GET","status_code":400,"body_size":51,"path":"/v1/finance/currconv?from=EUR&to=USD","latency":"45.719µs","time":"2022-01-03T07:43:11Z"}
```

## Clean up & refactor

If we take a look at our current folder with the latest changes we see a mix of
things:

```terminal
.
├── apierror    # internal stuff
├── finance     # functionalities for finance
├── middleware  # internal stuff
├── programming # functionalities for programming
├── ...
└── README.md
```

Our packages implementing real end user functionality are mixed up with internal
packages like the `apierror` and `middleware`.

To have a clearer separation we are going to refactor a bit our code. The idea
is to move internal packages to a folder named `internal` and the feature
packages to the `pkg` folder.

The final structure we want to achieve is:

```terminal
.
├── internal
│   ├── apierror
│   └── middleware
├── pkg
│   ├── finance
│   └── programming
├── ...
└── README.md
```

Let's make the changes:

```sh
mkdir internal
mkdir pkg
mv apierror internal
mv middleware internal
mv finance pkg
mv programming pkg
```

Next fix the all import errors and run the tests:

```sh
go test ./...
```

## Wrap up

Commit and push everything. Create a new tag.

```sh
git add .
git commit -m "refactor: add structure logger to gin"
git push
git tag -a v0.0.8 -m "v0.0.8"
git push origin v0.0.8
```