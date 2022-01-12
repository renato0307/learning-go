# Add programming/uuid to the CLI

Before adding the first command lets define the structure to accommodate all
functions available in the API.

We currently have the following functions:
* programming - uuid
* programming - jwt debugger
* finance - currency converter

Our CLI will allow us to call these functions using the following commands.

```sh
learning-go-cli programming uuid
```

Where:
* `learning-go-cli` - the app name
* `programming` - a command
* `uuid` - a sub-command of the `programming` command
* `api-token` - a flag to define the API security token

```sh
learning-go-cli programming jwtdebugger JWT
```

Where:
* `learning-go-cli` - the app name
* `programming` - a command
* `jwtdebugger` - a sub-command of the `programming` command
* `JWT` - an arg of the `jwtdebugger` sub-command

```sh
learning-go-cli finance currconv 10 --from EUR --to USD 
```

Where:
* `learning-go-cli` - the app name
* `finance` - a command
* `currconv` - a sub-command of the `finance` command
* `10` - an argument of the `currconv` sub-command
* `from` - a flag to define the source currency
* `to` - a flag to define the destination currency

As we did before we'll create a sub-folder for each category, where all commands
for that category can be implemented.

So, first let's create the `programming` folder inside the `cmd` folder:

```sh
mkdir cmd/programming
```

## Add the `programming` command

Next we need to implement the `programming` command. This command will have no
real functionality, its just an aggregator for all functions of this category.

We are not going to use command generator included in Cobra as it will generate
more stuff then we really need. 

Let's add the following content to the `cmd/programming/programming.go file`:

```go
package programming

import (
	"fmt"

	"github.com/spf13/cobra"
)

// NewProgrammingCmd represents the programming command
func NewProgrammingCmd() *cobra.Command {
	return &cobra.Command{
		Use:   "programming",
		Short: "Programming tools",
		Long:  ``,
		RunE: func(cmd *cobra.Command, args []string) error {
			return Execute(cmd, args)
		},
	}
}

// Execute implements all the logic associated with this command.
// In this case as it is an aggregation command will return an error
func Execute(cmd *cobra.Command, args []string) error {
	return fmt.Errorf("must specify a subcommand")
}

```

So basically we create a new command and implement the `RunE` function. This
function allows to return an error. We are using it because we want to return
an error if this command is executed directly without a sub-command.

For the `programming` command be available it must be associated to the root
command defined in the `root.go` file. To do that we need to change the `init`
function.

The code to add to the `root.go` file is:

```go

// ..

func init() {
	cobra.OnInitialize(initConfig)
	rootCmd.AddCommand(NewConfigureCommand())

	rootCmd.AddCommand(programming.NewProgrammingCmd()) // new
}
```

## Unit tests for the `programming` command

The contents of the `programming_test.go` file are:

```go
package programming

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestNewProgrammingCmd(t *testing.T) {
	// act
	cmd := NewProgrammingCmd()

	// assert
	assert.Equal(t, "programming", cmd.Use)
	assert.NotEmpty(t, cmd.Short, "Short description cannot be empty")
	assert.NotEmpty(t, cmd.Long, "Long description cannot be empty")
	assert.NotNil(t, cmd.RunE, "The RunE function must be defined")
}

func TestExecute(t *testing.T) {
	// arrange
	cmd := NewProgrammingCmd()

	// act
	err := execute(cmd, []string{})

	// assert
	assert.Error(t, err)
}
```

## Add the `uuid` sub-command

The `uuid` sub-command 

## Refactor the `uuid` command for better testability

The commands to implement will in most of the cases write output to the console.
Therefore, to be able to appropriately unit test the commands, we need to remove
that direct dependency so we can write the output to a buffer and inspect that
buffer to check for the appropriate behavior.

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

// Fprint knows how to print using an IOStreams struct
func (iostreams *IOStreams) Fprint(v interface{}) error {
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

Next we need to change the `uuid.go`, the `programming.go` and the `root.go`
files to use the `iostreams` package.

First let's do the changes in `programming.go`:

```go
```