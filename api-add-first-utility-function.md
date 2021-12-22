# Adding the first utility function to the API

For the same reasons the API will be organized using a similar approach of 
the library.

The categories will also be part of the URL, for example:

```sh
POST /programming/uuid                      # generates an uuid
POST /programming/base64/encode             # encodes a string as base64
POST /finance/calculate-compound-interests  # calculates interests
# and so on...
```

In the main folder we are going to create the `main.go` file which starts
the web server and create a folder for each category (programming, finance,
etc.)

The API will import the library and add the needed logic to handle HTTP
requests.

To better organize routes in the API we can use a Gin feature called
route grouping. Please check a simple example 
[here](https://github.com/gin-gonic/gin#grouping-routes).


## The code to generate UUIDs

So the first step is to create the `programming` folder and the files for 
UUID utility support.


```sh
mkdir programming
echo "package programming" > programming/programming.go
echo "package programming" > programming/programming_test.go
```

The code for `programming.go` is:

```go
package programming

import (
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/renato0307/learning-go-lib/programming"
)

// SetRouterGroup defines all the routes for the programming functions
func SetRouterGroup(base *gin.RouterGroup) *gin.RouterGroup {
	programmingGroup := base.Group("/programming")
	{
		programmingGroup.POST("/uuid", postUuid())
		// Add here more functions in the programming category
	}

	return programmingGroup
}

// postUuid handles the uuid request.
// It returns 200 on success.
// Reads the "no-hyphens" parameter from the query string to support
// UUIDs without hyphens.
func postUuid() gin.HandlerFunc {
	return func(c *gin.Context) {
		noHyphensParamValue := c.Query("no-hyphens")
		withoutHyphens := noHyphensParamValue == "true"

		output := programming.NewUuid(withoutHyphens)

		c.JSON(http.StatusOK, output)
	}
}
```

Let's break it down.

The `SetRouterGroup` function defines all the endpoints for the programming utilities.
Once the server receives the `POST /programming/uuid` request, it will be
processed by the function returned by `postUuid`.

The `postUuid` function, returns an another function (in this case,
an anonymous function) that complies with the `HandlerFunc` type interface:

```go
type HandlerFunc func(*Context)
```

The `Context` gives access to the HTTP request to get the query parameters,
for example.

After generating the UUID by calling the `NewUuid` function from the 
`programming` package, the `c.JSON` serializes the return status and the output
to the HTTP response.

## Changes on the main.go

To make everything work, we need the following changes in the `main.go` file:

```go
func main() {
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "Hello, welcome to the learning-go-api",
		})
	})

	base := r.Group("/v1")             // new
	programming.SetRouterGroup(base)   // new
	// finance.SetRouterGroup(base)    // for the future

	r.Run()
}
```

We define the API version by using *URI Versioning* (the `/v1` part).

The final result is `/v1/programming/uuid` route being added to the Gin engine.

## Manual testing

Go to the command line and run

```sh
go run main.go
```

In another terminal use `curl` or `httpie` to execute the call:

```sh
http POST localhost:8080/v1/programming/uuid
```

The result should be similar to:

```
HTTP/1.1 200 OK
Content-Length: 38
Content-Type: application/json; charset=utf-8
Date: Tue, 21 Dec 2021 18:18:56 GMT

"091650b9-8b50-4970-b5ac-7b511a55518b"
```

## Unit testing

The contents of the `programming_test.go` file are:

```go
package programming

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/gin-gonic/gin"
	"github.com/stretchr/testify/assert"
)

func setupGin() *gin.Engine {
	r := gin.Default()
	v1 := r.Group("/v1")
	SetRouterGroup(v1)

	return r
}

func TestPostUuid(t *testing.T) {
	// arrange
	r := setupGin()
	w := httptest.NewRecorder()
	req, _ := http.NewRequest("POST", "/v1/programming/uuid", nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusOK)

	body := w.Body.String()
	assert.Len(t, body, 38)
	assert.Contains(t, body, "-")
}

func TestPostUuidWithNoHyphen(t *testing.T) {
	// arrange
	r := setupGin()
	w := httptest.NewRecorder()
	req, _ := http.NewRequest("POST", "/v1/programming/uuid?no-hyphens=true", nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusOK)

	body := w.Body.String()
	assert.Len(t, body, 34)
	assert.NotContains(t, body, "-")
}
```