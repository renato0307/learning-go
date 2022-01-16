# Unit tests in the API using mocks

When writing unit tests there are two main schools of thought:

- __Solitary unit tests__, where other classes that are called by your class
or function under test are substituted with _mocks_ or _stubs_
- __Sociable unit tests__, for tests that allow your class or function under
tests talking to real collaborators.

I agree with
[this perspective](https://martinfowler.com/articles/practical-test-pyramid.html#SociableAndSolitary),
that says:

> At the end of the day it's not important to decide if you go for solitary or
> sociable unit tests. Writing automated tests is what's important. Personally,
> __I find myself using both approaches all the time__. If it becomes awkward to
> use real collaborators I will use mocks and stubs generously. If I feel like
> involving the real collaborator gives me more confidence in a test I'll only
> stub the outermost parts of my service.

The tests we did in the last section are in fact *sociable unit tests* as we 
are testing the API functions but we did not _mock_ the library.

This is OK as the current library is code is very simple, without "awkward" 
dependencies.

Nevertheless, if calling the library would require, for example, the
setup of a database or an external system, it could be harder to implement unit
tests and I would prefer to mock the library.

With this in mind, in this section we will do the necessary code changes so we
can mock the library when implementing the unit tests for the API.

To do that we need first to revisit what are interfaces in Go.

As described in the [Tour of Go](https://go.dev/tour/methods/9):
> An interface type is defined as a set of method signatures.
>
> A value of interface type can hold any value that implements those methods.

And

> A type implements an interface by implementing its methods. There is no
> explicit declaration of intent, no "implements" keyword.

For example (taken from [here](https://go.dev/tour/methods/10)):

```go
package main

import "fmt"

type I interface {
	M()
}

type T struct {
	S string
}

// This method means type T implements the interface I,
// but we don't need to explicitly declare that it does so.
func (t T) M() {
	fmt.Println(t.S)
}
```

For us to be able to mock the library we need to define an interface. This 
allows us to have two different implementations, a real one and a _mock_.

## Creating the interface

Open vscode in the `learning-go-lib` project.

As each category will provide a different set of functions, we will create
an interface per category.

In our current implementation that's the `programming` category.

So, inside the `programming` folder add a new file named `interface.go` with
the following contents, to define the `NewUuid` as part of the package
interface:

```go
package programming

type Interface interface {
	NewUuid(withoutHyphen bool) string
}

type ProgrammingFunctions struct {
}
```

Next we need to change the `NewUuid` implementation to bind it to the 
`ProgrammingFunctions` struct. This makes this structure implement the
interface.

To make the struct implement the interface, we need to put it as `receiver` of 
the function.

Function definition without `receiver`:

```go
func NewUuid(withoutHyphen bool) string
```

The function with `receiver` is:

```go
func (pf *ProgrammingFunctions) NewUuid(withoutHyphen bool) string
```

This is the new code in the `uuid.go` file:

```go
package programming

import (
	"strings"

	"github.com/google/uuid"
)


// NewUuid generates an UUID with the possibility
// to remove the hyphens
func (pf *ProgrammingFunctions) NewUuid(withoutHyphen bool) string { // receiver added
	uuidWithHyphen := uuid.New()

	if withoutHyphen {
		return strings.Replace(uuidWithHyphen.String(), "-", "", -1)
	}

	return uuidWithHyphen.String()
}
```

In the `uuid_test.go` file we can now apply the changes and observe the
difference on how the library code will be used:

```go
package programming

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

// creates a instance of the structure to be used as a receiver
var pf ProgrammingFunctions = ProgrammingFunctions{}

func TestNewUuidWithHyphen(t *testing.T) {
	uuidWithHyphen := pf.NewUuid(false) // "pf" used as a receiver

	assert.Len(t, uuidWithHyphen, 36)
	assert.Contains(t, uuidWithHyphen, "-") 
}

func TestNewUuidWithoutHyphen(t *testing.T) {
	uuidWithHyphen := pf.NewUuid(true) // "pf" used as a receiver

	assert.Len(t, uuidWithHyphen, 32)
	assert.NotContains(t, uuidWithHyphen, "-")
}
```

## Creating the mock implementation

Mocks in Go need to be implemented, in contrast with `unittest.mock` from the
[Python standard library](https://docs.python.org/dev/library/unittest.mock.html),
which removes "the need to create a host of stubs throughout your test suite".

To facilitate the generation of the mocks we are going to use
[mockery](https://github.com/vektra/mockery), a mock code auto-generator for
Golang.

[Download](https://github.com/vektra/mockery/releases) and install mockery.

After execute the following command to generate the stubs:

```sh
mockery --all --inpackage --case snake
```

This will generate a new file named `mock_interface.go` in the `programming`
folder. The contents of this file are:

```go
// Code generated by mockery v1.0.0. DO NOT EDIT.

package programming

import mock "github.com/stretchr/testify/mock"

// MockInterface is an autogenerated mock type for the Interface type
type MockInterface struct {
	mock.Mock
}

// NewUuid provides a mock function with given fields: withoutHyphen
func (_m *MockInterface) NewUuid(withoutHyphen bool) string {
	ret := _m.Called(withoutHyphen)

	var r0 string
	if rf, ok := ret.Get(0).(func(bool) string); ok {
		r0 = rf(withoutHyphen)
	} else {
		r0 = ret.Get(0).(string)
	}

	return r0
}
```

I would like to highlight the following:

1. There is a new struct called `MockInterface`
1. This struct is the receiver of the mock implementation of the `NewUuid`
function

With this done, we need to commit, push and tag this new version of the
`learning-go-lib`:

```sh
git add .
git commit -m "refactor: introduced interfaces and mocks"
git push
git tag -a v0.0.2 -m "v0.0.2"
git push origin v0.0.2
```

Next we need to change the API.

## Changing the API

Open the `learning-go-api` project using vscode.

First we need to change the `go.mod` to require the new version of the lib.

```terminal
module github.com/renato0307/learning-go-api

go 1.17

require github.com/gin-gonic/gin v1.7.7

require (
	github.com/renato0307/learning-go-lib v0.0.2 // new version
	github.com/stretchr/testify v1.7.0
)

// ... continues
```

Next we need to change the implementation of the API. We want to make the
package `programming` to reference only interfaces instead of referencing
specific implementations.

The full version of the `programming.go` file is:

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
func SetRouterGroup(p programming.Interface, base *gin.RouterGroup) *gin.RouterGroup {
	programmingGroup := base.Group("/programming")
	{
		programmingGroup.POST("/uuid", postUuid(p))
		// Add here more functions in the programming category
	}

	return programmingGroup
}

// postUuid handles the uuid request.
//
// Reads the "no-hyphens" parameter from the query string to support
// UUIDs without hyphens.
//
// It returns 200 on success.
func postUuid(p programming.Interface) gin.HandlerFunc {
	return func(c *gin.Context) {
		noHyphensParamValue := c.Query("no-hyphens")
		withoutHyphens := noHyphensParamValue == "true"

		uuid := p.NewUuid(withoutHyphens)
		output := postUuidOutput{UUID: uuid}

		c.JSON(http.StatusOK, output)
	}
}
```

Let me highlight the changes (check the image below):

1. The `SetRouterGroup` methods gets receives the interface for the library
package
1. The interface is passed to the `postUuid` function
1. The `postUuid` receives the interface
1. The interface is used to call the `NewUuid` function

With this, the programming package _no longer references a specific
implementation_. It references only interfaces.

![High level overview](/assets/api-unit-tests-mocks-1.png)

Next we need to fix the `main.go`.

The changes are simple:
1. The real implementation is attached to the `ProgrammingFunctions` struct
1. We need to send an instance to the `SetRouterGroup`

```go
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/renato0307/learning-go-api/programming"
	programminglib "github.com/renato0307/learning-go-lib/programming" // new
)

func main() {
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "Hello, welcome to the learning-go-api",
		})
	})

	base := r.Group("/v1")

	p := programminglib.ProgrammingFunctions{} // new
	programming.SetRouterGroup(&p, base)       // change

	r.Run()
}

```

The only missing step is to use the `MockInterface` in the tests, instead of
a real implementation.

The most relevant changes in the `programming_test.go` file are:

1. The `setupGin` function must receive a `mockInterface` and pass it to the
`SetRouterGroup`
1. The test must instantiate the `mockInterface` to define and assert the
expectations
1. The `mockCall := mockInterface.On("NewUuid", false)` states 
the `NewUuid` function must be called with the `false` argument.
1. The `mockCall.Return("1ce44be5-fe68-46f7-a153-51c1c91a4ae4")` defines
the return value of the `NewUuid` call
1. The `mockInterface.AssertExpectations(t)` asserts the `NewUuid` method
was called correctly

```go
func setupGin(mockInterface *programminglib.MockInterface) *gin.Engine {
	r := gin.Default()
	v1 := r.Group("/v1")
	SetRouterGroup(mockInterface, v1)

	return r
}

func TestPostUuid(t *testing.T) {
	// arrange
	mockInterface := programminglib.MockInterface{}
	mockCall := mockInterface.On("NewUuid", false)
	mockCall.Return("1ce44be5-fe68-46f7-a153-51c1c91a4ae4")

	r := setupGin(&mockInterface)
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

	mockInterface.AssertExpectations(t)
}
```

🏋️‍♀️ __CHALLENGE__: fix the other test in the file and make everything green.