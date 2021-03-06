# Adding the UUID generator function to the Library

As described in the 
[What are we going to build?](intro-what-are-we-going-to-build.md) section,
our utilities are going to be organized into categories, like Programming
or Finance.

As the number of categories and utilities will grow with time, a proper project
structure is needed to ensure we keep the code organized and easy to 
maintain.

The typical project layout of a Go project can be found
at [https://github.com/golang-standards/project-layout]().

I want to highlight the following parts:

>`/cmd`
> Main applications for this project. The directory name for each application 
> should match the name of the executable you want to have (e.g., /cmd/myapp).

> `/pkg`
> Library code that's ok to use by external applications
> (e.g., /pkg/mypubliclib). Other projects will import these libraries
> expecting them to work, so think twice before you put something here.

The libraries will not have executables so we don't need the `cmd` folder.

Having a `pkg` folder is overkill for this case as we don't need other folders.
This would only add an additional indirection.

So the approach here is be to create a folder per category:
* `programming`
* `finance`
* etc.

As we will not have any application, we need to delete the previously created
`main.go` file:

```sh
rm main.go
```

We will start by the programming category so create let's the folder:

```sh
mkdir programming
```
The first utility function is the `uuid` generator, so run the following 
commands to create the needed Go files:

```sh
touch programming/uuid.go
touch programming/uuid_test.go
```

Notice that the package name must match the folder where the files are.

The `uuid.go` file will contain the logic.
The `uuid_test.go` file will contain the tests.

Add the following contents in the `uuid.go` file:

```go
package programming

import (
	"strings"

	"github.com/google/uuid"
)

// NewUuid generates an UUID with the possibility
// to remove the hyphens
func NewUuid(withoutHyphen bool) string {
	uuidWithHyphen := uuid.New()

	if withoutHyphen {
		return strings.Replace(uuidWithHyphen.String(), "-", "", -1)
	}

	return uuidWithHyphen.String()
}

```

We are using the `github.com/google/uuid` to generate UUIDs. The import 
statement can generate an error message:

```terminal
could not import github.com/google/uuid (no required module provides package "github.com/google/uuid")
```

You can solve this by opening the `go.mod` file and press `Run go mod tidy`.

![High level overview](/assets/lib-add-first-utility-function-1.png)

----
???????????????? __GO-EXTRA: Comments__ 

The `godoc` tool processes Go source files to extract documentation about the
contents of the package.

For functions the first sentence should be a one-sentence summary that starts
with the name being declared.

For example:

```go
// Compile parses a regular expression and returns, if successful,
// a Regexp that can be used to match against text.
func Compile(str string) (*Regexp, error) {
```

Check more about comments in the
[Effective Go documentation](https://go.dev/doc/effective_go#commentary).

----

The next step is to add the tests (you can do it first if you prefer TDD...):


```go
package programming

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestNewUuidWithHyphen(t *testing.T) {
	uuidWithHyphen := NewUuid(false)

	assert.Len(t, uuidWithHyphen, 36)
	assert.Contains(t, uuidWithHyphen, "-")
}
```

We are going to use `github.com/stretchr/testify` library to provide common
assertions and mocking capabilities.

The tests can be executed from vscode in two different ways:
1. Using the "Testing" activity in the "Activity Bar" (#1 and #2 in the image)
1. Directly in the test file (#3 in the image)

The result is presented in the "Output" panel (#4 in the image).

![High level overview](/assets/lib-add-first-utility-function-2.png)

As an alternative the command line can also be used:

```sh
go test ./...
```

To finish, we need to complement the tests to ensure full coverage:

```go
package programming

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestNewUuidWithHyphen(t *testing.T) {
	uuidWithHyphen := NewUuid(false)

	assert.Len(t, uuidWithHyphen, 36)
	assert.Contains(t, uuidWithHyphen, "-")
}

func TestNewUuidWithoutHyphen(t *testing.T) {
	uuidWithHyphen := NewUuid(true)

	assert.Len(t, uuidWithHyphen, 32)
	assert.NotContains(t, uuidWithHyphen, "-")
}
```

The final results are (in this case using "Testing" from the Activity bar):

![High level overview](/assets/lib-add-first-utility-function-3.png)

To finish, commit and push all files to GitHub:

```sh
git add .
git commit -m "feat: add uuid function"
git push
```

To be able to import the library in the API we need to version it. This can be 
achieved by using tags.

Execute the following commands:

```sh
git tag -a v0.0.1 -m "v0.0.1"
git push origin v0.0.1
```

The result should be similar to:

```terminal
Enumerating objects: 1, done.
Counting objects: 100% (1/1), done.
Writing objects: 100% (1/1), 156 bytes | 156.00 KiB/s, done.
Total 1 (delta 0), reused 0 (delta 0)
To github.com:renato0307/learning-go-lib.git
 * [new tag]         v0.0.1 -> v0.0.1
```

# Next
 
The next section is
[Start the project for the API](it1-api-start-the-project.md).