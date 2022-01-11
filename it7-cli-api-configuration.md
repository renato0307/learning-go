# Add configuration to the CLI

All commands will require API authentication. To avoid having to enter the
credentials all the time we will make a command just to store the API
authentication information in a file so all commands can use them.

Reading and writing configurations when using Cobra can be easily done using
[Viper](https://github.com/spf13/viper).

So we will create a command named `configure` to achieve this purpose.

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
	"os"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
)

const (
	ClientIdFlag      string = "client-id"
	ClientSecretFlag  string = "client-secret"
	APIEndpointFlag   string = "api-endpoint"
	TokenEndpointFlag string = "token-endpoint"
)

// NewConfigureCommand creates the the configure command
func NewConfigureCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "configure",
		Short: "Configures the CLI",
		Long:  `Allows to define the API endpoints and the client credentials`,
		RunE: func(cmd *cobra.Command, args []string) error {
			return execute(cmd, args)
		},
	}

	cmd.Flags().StringP(ClientIdFlag,
		"c",
		"",
		"the client id to call the API")
	cmd.MarkFlagRequired(ClientIdFlag)

	cmd.Flags().StringP(ClientSecretFlag,
		"s",
		"",
		"the client secret to call the API")
	cmd.MarkFlagRequired(ClientSecretFlag)

	cmd.Flags().StringP(APIEndpointFlag,
		"a",
		"",
		"the API endpoint")
	cmd.MarkFlagRequired(APIEndpointFlag)

	cmd.Flags().StringP(TokenEndpointFlag,
		"t",
		"",
		"the endpoint to get authentication tokens")
	cmd.MarkFlagRequired(TokenEndpointFlag)

	return cmd
}

// execute implements all the logic associated with this command.
func execute(cmd *cobra.Command, args []string) error {
	clientId, _ := cmd.Flags().GetString(ClientIdFlag)
	viper.Set(ClientIdFlag, clientId)

	clientSecret, _ := cmd.Flags().GetString(ClientSecretFlag)
	viper.Set(ClientSecretFlag, clientSecret)

	apiEndpoint, _ := cmd.Flags().GetString(APIEndpointFlag)
	viper.Set(APIEndpointFlag, apiEndpoint)

	tokenEndpoint, _ := cmd.Flags().GetString(TokenEndpointFlag)
	viper.Set(TokenEndpointFlag, tokenEndpoint)

	return viper.WriteConfig()
}
```

I would like to highlight the following:

1. In the `NewConfigureCommand` function, besides the creation of the command,
we are also defining the four required flags.
1. In the `execute` function we read the flags value and save them to a config
file using `viper`.

For this to work we must do some changes in the `root.go` file:

```go
// ...

func Execute() {
	cobra.CheckErr(rootCmd.Execute())
}

func init() {
	cobra.OnInitialize(initConfig)            // new
	rootCmd.AddCommand(NewConfigureCommand()) // new
}
```

The `cobra.OnInitialize` will call the `initConfig` function before each command
is executed. We add the `configure` command to the `root` command. This makes
the `configure` command available automatically.

The `initConfig` will be implemented in the `configure.go` file as it is 
highly related with configuration.

The code to be added to the `configure.go` file is:

```go
// ...

// initConfig reads in config file and ENV variables if set
func initConfig() {

	// find home directory.
	home, err := os.UserHomeDir()
	cobra.CheckErr(err)

	// search config in home directory with name
	// ".learning-go-cli" (without extension).
	ext := "yaml"
	name := ".learning-go-cli"
	viper.AddConfigPath(home)
	viper.SetConfigType(ext)
	viper.SetConfigName(name)

	// creates config file if it does not exist
	_, err = createConfigFile(home, name, ext)
	cobra.CheckErr(err)

	// if a config file is found, read it in.
	if err := viper.ReadInConfig(); err == nil {
		cobra.CheckErr(err)
	}
}

// createConfigFile creates the config file if it does not exist
func createConfigFile(home string, name string, ext string) (string, error) {
	fileName := fmt.Sprintf("%s/%s.%s", home, name, ext)
	_, err := os.Stat(fileName)
	if os.IsNotExist(err) {
		file, err := os.Create(fileName)
		if err != nil {
			defer file.Close()
			return fileName, nil
		}
		return fileName, err
	}
	return fileName, nil
}
```

The `initConfig` will read the configuration from the
`$HOME/.learning-go-cli.yaml` file. If the configuration file will created, if
does not exist.

## Unit testing

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

The contents of the `configure_test.go` file are:

```go
package cmd

import (
	"fmt"
	"math/rand"
	"os"
	"testing"

	"github.com/spf13/viper"
	"github.com/stretchr/testify/assert"
)

func createFakeConfigFile(t *testing.T) {
	home, _ := os.UserHomeDir()
	ext := "yaml"
	name := fmt.Sprintf(".learning-go-cli-test-%d", rand.Uint64())
	fileName, err := createConfigFile(home, name, ext)
	if err != nil {
		assert.FailNow(t, "error creating config file")
	}
	defer os.Remove(fileName)
	viper.Reset()
	viper.AddConfigPath(home)
	viper.SetConfigType(ext)
	viper.SetConfigName(name)
}

func TestNewConfigureCommand(t *testing.T) {

	// act
	cmd := NewConfigureCommand()

	// assert
	assert.Equal(t, "configure", cmd.Use)
	assert.NotEmpty(t, cmd.Short, "Short description cannot be empty")
	assert.NotEmpty(t, cmd.Long, "Long description cannot be empty")
	assert.NotNil(t, cmd.RunE, "The RunE function must be defined")
	assert.NotNil(t, cmd.Flags().Lookup(ClientIdFlag))
	assert.NotNil(t, cmd.Flags().Lookup(ClientSecretFlag))
	assert.NotNil(t, cmd.Flags().Lookup(APIEndpointFlag))
	assert.NotNil(t, cmd.Flags().Lookup(TokenEndpointFlag))
}

func TestExecute(t *testing.T) {

	// arrange
	cmd := NewConfigureCommand()
	createFakeConfigFile(t)

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
	assert.Equal(t, "fake-c", viper.GetViper().Get(ClientIdFlag))
	assert.Equal(t, "fake-s", viper.GetViper().Get(ClientSecretFlag))
	assert.Equal(t, "fake-a", viper.GetViper().Get(APIEndpointFlag))
	assert.Equal(t, "fake-t", viper.GetViper().Get(TokenEndpointFlag))
}

func TestCreateConfigFile(t *testing.T) {

	// arrange
	home, _ := os.UserHomeDir()
	ext := "yaml"
	name := fmt.Sprintf(".learning-go-cli-test-%d", rand.Uint64())

	// act
	fileName, err := createConfigFile(home, name, ext)

	// assert
	if err != nil {
		assert.Fail(t, "error creating config file")
	}
	defer os.Remove(fileName)
	assert.Equal(t, fmt.Sprintf("%s/%s.%s", home, name, ext), fileName)
}
```

## Manual test

If we run:

```sh
go run main.go
```

We will see a new `configure` command in the list:

```
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

```
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

```
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