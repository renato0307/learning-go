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
the web server. We will create a folder for each category (programming, finance,
etc.)

The API will import the library and add the needed logic to handle HTTP
requests.

To better organize routes in the API we can use a Gin feature called
route grouping. Please check a simple example 
[here](https://github.com/gin-gonic/gin#grouping-routes).


## The implementation of the web service

So the first step is to create the `programming` folder and the implementation
and test files.


```sh
mkdir programming
touch programming/programming.go
touch programming/programming_test.go
```

The code for `programming.go` is:

```go
package programming

import (
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/renato0307/learning-go-lib/programming"
)

// postUuidOutput is the output of the "POST /programming/uuid" action
type postUuidOutput struct {
	UUID string `json:"uuid"`
}

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

		uuid := programming.NewUuid(withoutHyphens)
		output := postUuidOutput{UUID: uuid}

		c.JSON(http.StatusOK, output)
	}
}
```

Let's break it down.

The `postUuidOutput` struct defines the structure of the web service response.

The `SetRouterGroup` function defines all the endpoints for the programming
utilities. Once the server receives the `POST /programming/uuid` request, it
will be processed by the function returned by `postUuid`.

The `postUuid` function, returns another function (in this case, an anonymous
[closure](https://go.dev/tour/moretypes/25) function) that must comply with the
`HandlerFunc` type interface:

```go
type HandlerFunc func(*Context)
```

The `Context` gives access to the HTTP request, for example to get the query
parameters

After generating the UUID by calling the `NewUuid` function from the 
`programming` package imported the library, the `c.JSON` serializes the return
status and the output to the HTTP response.

----
🕵️‍♀️ __GO-EXTRA: Struct Fields Meta-data & JSON__

A struct in Go allows adding meta-data to its fields.

The format for attaching meta-data is:

```go
type strutName struct {
   fieldName type `key:value key2:value2 key3:value3`
}
```

A common use for the meta-data is for JSON operations, like `Marshal`.

We are using this for the output structures, specifying the name of the field
when converting from and to JSON. In the example bellow, the `UUID` field will
have the `uuid` name when transformed to and from JSON.

```go
type postUuidOutput struct {
	UUID string `json:"uuid"`
}
```

For more information about JSON and Go check
[this blog post](https://go.dev/blog/json).

----

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


## Unit testing

The tests go to the `programming_test.go` file.

The first test will cover the execution with hyphens:

```go
func TestPostUuid(t *testing.T) {
	// arrange
	r := gin.Default()
	v1 := r.Group("/v1")
	SetRouterGroup(v1)

	w := httptest.NewRecorder()
	req, _ := http.NewRequest("POST", "/v1/programming/uuid", nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusOK)

	output := postUuidOutput{}
	err := json.Unmarshal(w.Body.Bytes(), &output)

	assert.Nil(t, err)
	assert.Len(t, output.UUID, 36)
	assert.Contains(t, output.UUID, "-")
}
```

The HTTP based testing use the `net/http/httptest` package, which allows to
record the result of the request so we can make assertions over it.

In the `arrange` block:
1. Initialize Gin and the routes
1. Create the HTTP recorder and the request to execute

The `ServeHTTP` function executes a request and writes to the response.

In the `assert`block:
1. Check the return status
1. Confirm we receive an UUID with hyphens

After we also need to add a test for the case _without hyphens_.

🏋️‍♀️ __CHALLENGE__: don't scroll down and try to do this test by yourself!

The final contents of the `programming_test.go` file is:

```go
package programming

import (
	"encoding/json"
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

	output := postUuidOutput{}
	err := json.Unmarshal(w.Body.Bytes(), &output)

	assert.Nil(t, err)
	assert.Len(t, output.UUID, 36)
	assert.Contains(t, output.UUID, "-")
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

	output := postUuidOutput{}
	err := json.Unmarshal(w.Body.Bytes(), &output)

	assert.Nil(t, err)
	assert.Len(t, output.UUID, 32)
	assert.NotContains(t, output.UUID, "-")
}
```

## Manual testing

Go to the command line and run

```sh
go run main.go
```

In another terminal use `httpie` to execute the call:

```sh
http POST localhost:8080/v1/programming/uuid
```

The result should be similar to:

```
HTTP/1.1 200 OK
Content-Length: 47
Content-Type: application/json; charset=utf-8
Date: Wed, 21 Dec 2021 21:02:37 GMT

{
    "uuid": "2ea3a39b-51a1-4fe3-80b0-9d9a33d176be"
}
```