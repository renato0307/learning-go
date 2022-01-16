# Add the programming/uuid command to the CLI

Before adding the first command that uses the learning-go-api let's define the
structure to accommodate all functions available.

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

Let's add the following content to the `cmd/programming/programming.go` file:

```go
package programming

import (
	"fmt"

	"github.com/renato0307/learning-go-cli/internal/config"
	"github.com/renato0307/learning-go-cli/internal/iostreams"
	"github.com/spf13/cobra"
)

// NewProgrammingCmd represents the programming command
func NewProgrammingCmd(iostreams *iostreams.IOStreams) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "programming",
		Short: "Programming tools",
		Long:  `Provides several programming tools like uuid generation, etc.`,
		RunE:  executeProgramming(),
	}

	return cmd
}

// executeProgramming implements all the logic associated with this command.
// In this case as it is an aggregation command will return an error
func executeProgramming() func(cmd *cobra.Command, args []string) error {
	return func(cmd *cobra.Command, args []string) error {
		return fmt.Errorf("must specify a subcommand")
	}
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
	cobra.OnInitialize(config.InitConfig)

	iostreams := &iostreams.IOStreams{Out: os.Stdout}

	rootCmd.AddCommand(NewConfigureCommand(iostreams))

	programmingCmd := programming.NewProgrammingCmd(iostreams)   // new
	config.AddCommandWithConfigPreCheck(rootCmd, programmingCmd) // new
}
```

As the `programming` command requires calling the API, we add it using the
`config.AddCommandWithConfigPreCheck` function we implemented in the building
blocks to ensure the right configurations exist.

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
	cmd := NewProgrammingCmd(nil)

	// assert
	assert.Equal(t, "programming", cmd.Use)
	assert.NotEmpty(t, cmd.Short, "Short description cannot be empty")
	assert.NotEmpty(t, cmd.Long, "Long description cannot be empty")
	assert.NotNil(t, cmd.RunE, "The RunE function must be defined")
}

func TestExecute(t *testing.T) {
	// arrange
	cmd := NewProgrammingCmd(nil)

	// act
	err := cmd.Execute()

	// assert
	assert.Error(t, err)
}
```

## Add the `uuid` sub-command

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

I'll provide a little guidance:

1. Create a function to create the sub-command
1. Create a function to execute the sub-command
1. Implement the sub-command logic, that needs to
   * Get a new Access Token
   * Execute the request to the API
   * Handle errors and parse the result
   * Print the result as indented JSON
1. Add the sub-command to the `programming` command.

The following files are needed:

```sh
touch internal/programming/uuid.go
touch internal/programming/uuid_test.go
```

The function used to create the sub-command is (put it in the `uuid.go` file):

```go
const NoHyphensFlag string = "no-hyphens"

// NewProgrammingCmd represents the programming command
func NewProgrammingUuidCmd(iostreams *iostreams.IOStreams) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "uuid",
		Short: "Generates an UUID",
		Long:  `Generates an UUID, with or without hyphens.`,
		RunE:  executeProgrammingUuid(iostreams),
	}

	cmd.Flags().Bool(NoHyphensFlag,
		false,
		"if set the UUID generated will not contains hyphens")

	return cmd
}
```

The contents of the `executeProgrammingUuid` are:

```go
// executeProgrammingUuid implements all the logic associated with this command.
// In this case as it is an aggregation command will return an error
func executeProgrammingUuid(iostreams *iostreams.IOStreams) func(cmd *cobra.Command, args []string) error {
	return func(cmd *cobra.Command, args []string) error {

		// creates the base request
		apiEndpoint := config.GetString(config.APIEndpointFlag)
		realUrl := fmt.Sprintf("%s/programming/uuid", apiEndpoint)
		request, err := http.NewRequest("POST", realUrl, nil)
		if err != nil {
			return fmt.Errorf("error creating the request to call the API: %w", err)
		}

		// handles the "no-hyphens" flag
		noHyphens, err := cmd.Flags().GetBool(NoHyphensFlag)
		if err != nil {
			return err
		}
		if noHyphens {
			q := request.URL.Query()
			q.Add("no-hyphens", "true")
			request.URL.RawQuery = q.Encode()
		}

		// adds authentication
		token, err := auth.NewAccessToken()
		if err != nil {
			return fmt.Errorf("error getting the JWT to call the API: %w", err)
		}
		request.Header = map[string][]string{
			"Authentication": {token.AccessToken},
		}

		// calls API and reads response
		response, err := http.DefaultClient.Do(request)
		if err != nil {
			return fmt.Errorf("error calling the API: %w", err)
		}
		defer response.Body.Close()

		if response.StatusCode != http.StatusOK {
			apiError, err := ioutil.ReadAll(response.Body)
			if err != nil {
				return fmt.Errorf("error parsing API error: %w", err)
			}

			err = errors.New(string(apiError))
			return fmt.Errorf("error calling the API: %w", err)
		}

		uuid, err := ioutil.ReadAll(response.Body)
		if err != nil {
			return fmt.Errorf("error reading the UUID: %w", err)
		}

		// parse and print response as indented JSON
		var anyJson map[string]interface{}
		err = json.Unmarshal(uuid, &anyJson)
		if err != nil {
			return fmt.Errorf("parsing API response: %w", err)
		}
		output, _ := json.MarshalIndent(anyJson, "", "  ")

		_, err = fmt.Fprintln(iostreams.Out, string(output))
		if err != nil {
			return fmt.Errorf("error writing to the output: %w", err)
		}

		return nil
	}
}
```

To complete the implementation we need to add the new sub-command to the 
`programming` command. So, in the `programming.go` file do the following
change:

```go
//...

// NewProgrammingCmd represents the programming command
func NewProgrammingCmd(iostreams *iostreams.IOStreams) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "programming",
		Short: "Programming tools",
		Long:  `Provides several programming tools like uuid generation, etc.`,
		RunE:  executeProgramming(),
	}

	config.AddCommandWithConfigPreCheck(cmd, NewProgrammingUuidCmd(iostreams)) // new

	return cmd
}

// ...
```

## Unit testing

To implement the unit tests we need to use fake servers for the API and the 
authentication OAuth2 endpoint. 

In the test we will:

1. Use the mock helper functions we defined before
1. Define test cases in a list and run them all using a loop like we did before

The mock helper functions will return an `httptest.Server`. We will override
the configurations using the `config.Set` function to use the fake endpoints.

To code of the `uuid_test.go` file is:

```go
package programming

import (
	"bytes"
	"fmt"
	"net/http"
	"testing"

	"github.com/renato0307/learning-go-cli/internal/config"
	"github.com/renato0307/learning-go-cli/internal/iostreams"
	"github.com/renato0307/learning-go-cli/internal/testhelpers"
	"github.com/stretchr/testify/assert"
)

func TestNewProgrammingUuidCmd(t *testing.T) {
	// arrange
	buffer := &bytes.Buffer{}
	iostreams := &iostreams.IOStreams{Out: buffer}

	// act
	cmd := NewProgrammingUuidCmd(iostreams)

	// assert
	assert.Equal(t, "uuid", cmd.Use)
	assert.NotEmpty(t, cmd.Short, "Short description cannot be empty")
	assert.NotEmpty(t, cmd.Long, "Long description cannot be empty")
	assert.NotNil(t, cmd.RunE, "The RunE function must be defined")
	assert.NotNil(t, cmd.Flags().Lookup(NoHyphensFlag))
}

func TestExecuteProgrammingUuid(t *testing.T) {

	uuid := "da308fbd-cba9-485a-b4c1-6677aaa732a4"
	uuidNoHyphens := "da308fbdcba9485ab4c16677aaa732a4"

	testCases := []struct {
		ApiStatusCode     int
		ApiResponse       string
		ApiExpectedParams []string
		OutputContains    string
		Args              []string
		ErrorNil          bool
		Purpose           string
	}{
		{
			ApiStatusCode:  http.StatusOK,
			ApiResponse:    fmt.Sprintf("{\"uuid\": \"%s\"}", uuid),
			Args:           []string{},
			OutputContains: uuid,
			ErrorNil:       true,
			Purpose:        "success case",
		},
		{
			ApiStatusCode:     http.StatusOK,
			ApiResponse:       fmt.Sprintf("{\"uuid\": \"%s\"}", uuidNoHyphens),
			ApiExpectedParams: []string{"no-hyphens"},
			OutputContains:    uuidNoHyphens,
			Args:              []string{fmt.Sprintf("--%s", NoHyphensFlag)},
			ErrorNil:          true,
			Purpose:           "success case with no hyphens",
		},
		{
			ApiStatusCode: http.StatusBadRequest,
			ApiResponse:   "{\"message\": \"request is malformed\"}",
			Args:          []string{},
			ErrorNil:      false,
			Purpose:       "api returns error",
		},
		{
			ApiStatusCode: http.StatusOK,
			ApiResponse:   "something that is not a valid json",
			Args:          []string{},
			ErrorNil:      false,
			Purpose:       "error on invalid json",
		},
	}

	for _, tc := range testCases {
		// arrange
		buffer := &bytes.Buffer{}
		iostreams := &iostreams.IOStreams{Out: buffer}
		cmd := NewProgrammingUuidCmd(iostreams)

		tokenSrv := testhelpers.NewAuthTestServer()
		defer tokenSrv.Close()
		config.Set(config.TokenEndpointFlag, tokenSrv.URL)

		apiSrv := testhelpers.NewAPITestServer(
			tc.ApiResponse,
			tc.ApiExpectedParams,
			tc.ApiStatusCode)
		defer apiSrv.Close()
		config.Set(config.APIEndpointFlag, apiSrv.URL)

		// act
		cmd.SetArgs(tc.Args)
		err := cmd.Execute()

		// assert
		if tc.ErrorNil {
			assert.NoError(t, err, "error found for "+tc.Purpose)
			assert.Contains(t,
				buffer.String(),
				tc.OutputContains,
				"output is for right for "+tc.Purpose)
		} else {
			assert.Error(t, err, "error not found for "+tc.Purpose)
		}
	}
}
```

## Manual testing

To execute the `uuid` command please first run `configure` command.

Example:

```sh
go run main.go configure \
        -a https://localhost:8080/v1 \
        -c your_client_id \
        -s your_client_secret \
        -t https://learning-go-renato0307.auth.eu-west-1.amazoncognito.com/oauth2/token
```

Also start the API locally or use the one running on the local k8s cluster.

After that execute the following command to see the help:

```sh
go run main.go programming uuid --help
```

The output should be similar to:

```terminal
Generates an UUID, with or without hyphens.

Usage:
  learning-go-cli programming uuid [flags]

Flags:
  -h, --help         help for uuid
      --no-hyphens   if set the UUID generated will not contains hyphens
```

To generate an UUID run:

```sh
go run main.go programming uuid
```

The output should be similar to:

```json
{
  "uuid": "4819c2aa-7028-46f7-adf6-e5e95e9ffae9"
}
```

Try also to run it with the `--no-hyphens` flag:

```sh
go run main.go programming uuid --no-hyphens
```

The output should be similar to:

```json
{
  "uuid": "1c91273c8cf64c699ad57d8c3cd16f63"
}
```

## Wrap up

Commit and push everything.

```sh
git add .
git commit -m "feat: add programming/uuid command"
git push
```