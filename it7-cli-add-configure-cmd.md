# Add the configure command to the CLI

The `configure` command will store the authentication flags in a file so they
can be reused in all commands that call the API.

The command will have the following flags:

* `client-id` - sets the Cognito client id
* `client-secret` - sets the Cognito client secret
* `token-endpoint` - sets the Cognito OAuth2 token endpoint
* `api-endpoint` - sets the endpoint for the learning-go-api

Let's start by creating needed files in the `cmd` folder.

```sh
touch cmd/configure.go
touch cmd/configure_test.go
```

The contents of the `configure.go` file are:

```go
package cmd

import (
	"fmt"

	"github.com/renato0307/learning-go-cli/internal/config"
	"github.com/renato0307/learning-go-cli/internal/iostreams"
	"github.com/spf13/cobra"
)

// NewConfigureCommand creates the the configure command
func NewConfigureCommand(iostreams *iostreams.IOStreams) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "configure",
		Short: "Configures the CLI",
		Long:  `Allows to define the API endpoints and the client credentials`,
		RunE:  executeConfigure(iostreams),
	}

	cmd.Flags().StringP(config.ClientIdFlag,
		"c",
		"",
		"the client id to call the API")
	cmd.MarkFlagRequired(config.ClientIdFlag)

	cmd.Flags().StringP(config.ClientSecretFlag,
		"s",
		"",
		"the client secret to call the API")
	cmd.MarkFlagRequired(config.ClientSecretFlag)

	cmd.Flags().StringP(config.APIEndpointFlag,
		"a",
		"",
		"the API endpoint")
	cmd.MarkFlagRequired(config.APIEndpointFlag)

	cmd.Flags().StringP(config.TokenEndpointFlag,
		"t",
		"",
		"the endpoint to get authentication tokens")
	cmd.MarkFlagRequired(config.TokenEndpointFlag)

	return cmd
}

// executeConfigure implements all the logic associated with this command.
func executeConfigure(iostreams *iostreams.IOStreams) func(cmd *cobra.Command, args []string) error {

	return func(cmd *cobra.Command, args []string) error {
		clientId, _ := cmd.Flags().GetString(config.ClientIdFlag)
		clientSecret, _ := cmd.Flags().GetString(config.ClientSecretFlag)
		apiEndpoint, _ := cmd.Flags().GetString(config.APIEndpointFlag)
		tokenEndpoint, _ := cmd.Flags().GetString(config.TokenEndpointFlag)

		err := config.WriteAuthenticationConfig(
			clientId,
			clientSecret,
			apiEndpoint,
			tokenEndpoint,
		)

		if err == nil {
			fmt.Fprintf(iostreams.Out, "configuration updated!")
		}
		return err
	}
}
```

I would like to highlight the following:

1. In the `NewConfigureCommand` function, besides the creation of the command,
we are also defining the four required flags.
1. In the `executeConfigure` function we read the flags value and save them to
a config file using a one of the functions we implemented before in the
`config` package.
1. In the `executeConfigure` function we print a success message using the
`iostreams.Out` writer instead of printing to the standard output.

For this to work we must do some changes in the `root.go` file:

```go
package cmd

import (
	"os"

	"github.com/renato0307/learning-go-cli/internal/config"
	"github.com/renato0307/learning-go-cli/internal/iostreams"
	"github.com/spf13/cobra"
)

// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
	Use:   "learning-go-cli",
	Short: "CLI for the learning-go-api",
	Long: `The learning-go-api provides with utility functions like UUID
generation, a currency converter, a JWT debugger, etc.`,
	Version: "0.0.1",
}

// Execute adds all child commands to the root command and sets flags
// appropriately. This is called by main.main(). It only needs to happen once
// to the rootCmd.
func Execute() {
	cobra.CheckErr(rootCmd.Execute())
}

func init() {
	cobra.OnInitialize(config.InitConfig) // new

	iostreams := &iostreams.IOStreams{Out: os.Stdout}  // new
	rootCmd.AddCommand(NewConfigureCommand(iostreams))  // new
}
```

The `cobra.OnInitialize` will make the `config.InitConfig` function be invoked
before each command is executed. We also add the `configure` command to the
`root` command.

## Unit testing

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

The contents of the `configure_test.go` file are:

```go
package cmd

import (
	"bytes"
	"os"
	"testing"

	"github.com/renato0307/learning-go-cli/internal/config"
	"github.com/renato0307/learning-go-cli/internal/iostreams"
	"github.com/stretchr/testify/assert"
)

func TestNewConfigureCommand(t *testing.T) {
	// act
	cmd := NewConfigureCommand(nil)

	// assert
	assert.Equal(t, "configure", cmd.Use)
	assert.NotEmpty(t, cmd.Short, "Short description cannot be empty")
	assert.NotEmpty(t, cmd.Long, "Long description cannot be empty")
	assert.NotNil(t, cmd.RunE, "The RunE function must be defined")
	assert.NotNil(t, cmd.Flags().Lookup(config.ClientIdFlag))
	assert.NotNil(t, cmd.Flags().Lookup(config.ClientSecretFlag))
	assert.NotNil(t, cmd.Flags().Lookup(config.APIEndpointFlag))
	assert.NotNil(t, cmd.Flags().Lookup(config.TokenEndpointFlag))
}

func TestExecuteConfigure(t *testing.T) {
	// arrange
	// arrange
	buffer := &bytes.Buffer{}
	iostreams := &iostreams.IOStreams{Out: buffer}
	cmd := NewConfigureCommand(iostreams)

	fileName := config.CreateFakeConfigFile(t)
	defer os.Remove(fileName)

	// act
	cmd.SetArgs([]string{
		"-c", "fake-c",
		"-s", "fake-s",
		"-a", "fake-a",
		"-t", "fake-t",
	})
	err := cmd.Execute()

	// assert
	assert.NoError(t, err)
	assert.Equal(t, "fake-c", config.GetString(config.ClientIdFlag))
	assert.Equal(t, "fake-s", config.GetString(config.ClientSecretFlag))
	assert.Equal(t, "fake-a", config.GetString(config.APIEndpointFlag))
	assert.Equal(t, "fake-t", config.GetString(config.TokenEndpointFlag))
	assert.Equal(t, "configuration updated!", buffer.String())
}
```

## Manual test

If we run:

```sh
go run main.go
```

We will see a new `configure` command in the list:

```terminal
The learning-go-api provides with utility functions like UUID
generation, a currency converter, a JWT debugger, etc.

Usage:
  learning-go-cli [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  configure   Configures the CLI
  help        Help about any command

Flags:
  -h, --help      help for learning-go-cli
  -v, --version   version for learning-go-cli

Use "learning-go-cli [command] --help" for more information about a command.
```

If we run:

```sh
go run main.go configure
```

We will get the help for the `configure` command:

```terminal
Error: required flag(s) "api-endpoint", "client-id", "client-secret", "token-endpoint" not set
Usage:
  learning-go-cli configure [flags]

Flags:
  -a, --api-endpoint string     the API endpoint
  -c, --client-id string        the client id to call the API
  -s, --client-secret string    the client secret to call the API
  -h, --help                    help for configure
  -t, --token-endpoint string   the endpoint to get authentication tokens

Error: required flag(s) "api-endpoint", "client-id", "client-secret", "token-endpoint" not set
exit status 1
```

Now, if we execute the `configure` command with all required flags:

```sh
go run main.go configure \
	-a fake_api_endpoint \
	-c fake_client_id \
	-s fake_secret_id \
	-t fake_token_endpoint
```

And then we `cat` the configuration file:

```sh
cat $HOME/.learning-go-cli.yaml
```

The contents should be:

```yaml
api-endpoint: fake_api_endpoint
client-id: fake_client_id
client-secret: fake_secret_id
token-endpoint: fake_token_endpoint
```

## Wrap up

Commit and push everything.

```sh
git add .
git commit -m "feat: add configure command"
git push
```