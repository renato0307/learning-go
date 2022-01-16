# Change API file structure

Don't know if you noticed but the file organization in the API is a bit 
different from the Library.


The API structure contains several functionalities inside the same file.

For example, the programming category contains UUID generation and the JWT
debugger, all implemented inside the `programming.go` file.


```terminal
.
├── finance
│   ├── finance.go          # => all functionalities in a single file
│   └── finance_test.go
├── programming
│   ├── programming.go      # => all functionalities in a single file
│   └── programming_test.go
├── ...
└── README.md

```

In the Library we have cerate a file per functionality:

```terminal
.
├── finance
│   ├── currconv.go       # => one functionality per file
│   ├── currconv_test.go
│   ├── interface.go
│   └── mock_interface.go
├── programming
│   ├── interface.go
│   ├── jwtdebugger.go    # => one functionality per file
│   ├── jwtdebugger_test.go
│   ├── mock_interface.go
│   ├── uuid.go           # => one functionality per file
│   └── uuid_test.go
├── ...
└── README.md
```

If we start growing the number of functionalities, the API Go files can become
quite large.

For this reason, before we continue with any other changes, let's refactor the
API to be organized like the Library.

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

Each category contains a common/centralized function, the `SetRouterGroup`.

This one we will keep inside the `finance.go` or `programming.go` file.

It's the specific functionality code that we need to change.

## Changing the finance package

So, inside the `finance.go` file we just leave the `SetRouterGroup` function:

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
```

In a similar fashion the `finance_test.go` file contains only generic functions:

```go
package finance

import (
	"github.com/gin-gonic/gin"
	financelib "github.com/renato0307/learning-go-lib/finance"
)

func setupGin(mockInterface *financelib.MockInterface) *gin.Engine {
	r := gin.Default()
	v1 := r.Group("/v1")
	SetRouterGroup(mockInterface, v1)

	return r
}

```

We must also create the `finance/currconv.go` and `finance/currconv_test.go` with
the rest of the code removed from the `finance/finance.go` and
`finance/finance_test.go`.

## Changing the programming package

Do the same changes for the `programming` package. In the end you should have
the following structure:

```terminal
.
├── finance
│   ├── currconv.go
│   ├── currconv_test.go
│   ├── finance.go
│   └── finance_test.go
├── programming
│   ├── jwtdebugger.go
│   ├── jwtdebugger_test.go
│   ├── programming.go
│   ├── programming_test.go
│   ├── uuid.go
│   └── uuid_test.go
├── ...
└── README.md
```

## Wrap up

Run the tests:

```sh
go test ./...
```

Commit and push everything. Create a new tag.

```sh
git add .
git commit -m "refactor: one file per functionality"
git push
git tag -a v0.0.5 -m "v0.0.5"
git push origin v0.0.5
```