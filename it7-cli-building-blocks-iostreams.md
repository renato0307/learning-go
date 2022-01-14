# CLI building blocks - IO streams & testing helpers

The commands to implement will in most of the cases write output to the console.
Therefore, to be able to appropriately unit test the commands, we need to remove
the dependency from `os.Stdout` so we can write the output to a buffer and
inspect that buffer to check for the appropriate behavior.

Other well known CLIs (like the GitHub CLI) use this approach.

Basically consists in having a struct with the input, output and error
readers/writers interfaces.

So, to do that, let's create a package called `iostreams`. As this is an
internal logic we will put it in the `internal` folders.

```sh
mkdir -p internal/iostreams
touch internal/iostreams/iostreams.go
touch internal/iostreams/iostreams_test.go
```

The contents of the `iostreams.go` is:

```go
package iostreams

import (
	"fmt"
	"io"
)

// IOStreams represents the structures needed for input/output in commands
// Currently only supports output.
type IOStreams struct {
	Out io.Writer
}

// PrintOutput knows how to print using an IOStreams struct
func (iostreams *IOStreams) Fprint(v interface{}) (n int, err error) {
	return fmt.Fprint(iostreams.Out, v)
}
```

Basically we have a structure with an `io.Writer` interface and a method to
print a value to the output.

Let's unit test this. The contents of the `iostreams_test.go` are:

```go
package iostreams

import (
	"bytes"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestPrintOutput(t *testing.T) {
	// arrange
	buffer := &bytes.Buffer{}
	iostreams := IOStreams{Out: buffer}
	s := "my test string"

	// act
	iostreams.Fprint(s)

	// assert
	assert.Equal(t, s, buffer.String())
}
```
## Additional helpers for testing

All commands will need a fake API and a fake OAuth2 token generation API.

We can create two functions for that.

Let's put them in the `internal/testhelpers` folder.

```sh
mkdir -p internal/testhelpers
touch internal/testhelpers/helpers.go
```

The helpers are:

```go
package testhelpers

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"

	"github.com/renato0307/learning-go-cli/internal/auth"
)

// NewAuthTestServer create an httptest.Server to test commands requiring
// API authentication
func NewAuthTestServer() *httptest.Server {
	srv := httptest.NewServer(
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// to test the code sends an authorization token
			if r.Header.Get("Authorization") == "" {
				w.WriteHeader(http.StatusBadRequest)
				w.Write([]byte("Unauthorized"))
				return
			}
			w.WriteHeader(http.StatusOK)
			body, _ := json.Marshal(auth.AccessToken{})
			w.Write(body)
		}))

	return srv
}

// NewAPITestServer create an httptest.Server to test commands requiring
// to call the API
func NewAPITestServer(body string, expectedQueryParams []string, httpStatus int) *httptest.Server {
	srv := httptest.NewServer(
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			for _, qp := range expectedQueryParams {
				if r.URL.Query().Get(qp) == "" {
					w.WriteHeader(http.StatusBadRequest)
					w.Write([]byte(fmt.Sprintf("parameter %s expected", qp)))
					return
				}
			}
			w.WriteHeader(httpStatus)
			w.Write([]byte(body))
		}))

	return srv
}
```