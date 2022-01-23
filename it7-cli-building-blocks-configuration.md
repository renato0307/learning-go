# CLI building blocks - configuration

All commands will require API authentication. To do that, we need to have a way
to pass the API endpoint, the client id and secrets, and the OAuth2 token
endpoint, to all commands.

 To avoid having to enter the credentials all the time or have the commands
 implement replicated logic we will create a Go package to handle
 configurations.

Additionally we will be reading and writing configurations using
[Viper](https://github.com/spf13/viper). The new package will also abstract
the rest of the code for Viper.

This package will include:

1. Definition of common flags used for authentication configuration
1. Viper initialization
1. Create configuration files
1. Create fake configuration files for testing purposes
1. Set and get configuration values
1. Load and assert configurations before each command


This package will be named `config` and placed in `internal/config`.

```sh
mkdir -p internal/config
touch internal/config/config.go
touch internal/config/config_test.go
```

The Viper initialization includes the setup and creation of the
configuration file. The configuration file will be 
`$HOME/.learning-go-cli.yaml`.

The code needed in `config.go` follows. By reading the function comments you
should be able to understand each function.

```go
package config

import (
	"fmt"
	"math/rand"
	"os"
	"testing"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
	"github.com/stretchr/testify/assert"
)

// Common flags
const (
	ClientIdFlag      string = "client-id"
	ClientSecretFlag  string = "client-secret"
	APIEndpointFlag   string = "api-endpoint"
	TokenEndpointFlag string = "token-endpoint"
)

// initConfig reads in config file and ENV variables if set
func InitConfig() {

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
	_, err = CreateConfigFile(home, name, ext)
	cobra.CheckErr(err)

	// if a config file is found, read it in.
	if err := viper.ReadInConfig(); err == nil {
		cobra.CheckErr(err)
	}
}

// CreateConfigFile creates the config file if it does not exist
func CreateConfigFile(home string, name string, ext string) (string, error) {
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

// CreateFakeConfigFile configures viper to write to a temporary file
func CreateFakeConfigFile(t *testing.T) string {
	home, _ := os.UserHomeDir()
	ext := "yaml"
	name := fmt.Sprintf(".learning-go-cli-test-%d", rand.Uint64())
	fileName, err := CreateConfigFile(home, name, ext)
	if err != nil {
		assert.FailNow(t, "error creating config file")
	}

	viper.Reset()
	viper.AddConfigPath(home)
	viper.SetConfigType(ext)
	viper.SetConfigName(name)

	Set(APIEndpointFlag, "fake_endpoint")
	Set(TokenEndpointFlag, "fake_endpoint")
	Set(ClientIdFlag, "fake_client_id")
	Set(ClientSecretFlag, "fake_client_secret")
	viper.WriteConfig()

	return fileName
}
```

Next, we need to handle functions to get, set and write configurations.

The code is:

```go

// GetString returns a configuration string
func GetString(key string) string {
	return viper.GetString(key)
}

// Set defines a configuration value
func Set(key string, value interface{}) {
	viper.Set(key, value)
}

// WriteAuthenticationConfig persists the authentication configuration
func WriteAuthenticationConfig(
	clientId,
	clientSecret,
	apiEndpoint,
	tokenEndpoint string) error {

	Set(ClientSecretFlag, clientSecret)
	Set(ClientIdFlag, clientId)
	Set(APIEndpointFlag, apiEndpoint)
	Set(TokenEndpointFlag, tokenEndpoint)

	return viper.WriteConfig()
}
```

When defining a command, we must ensure the configurations needed to call
the API are set.

We can take advantage of `PreRun` function supported by Cobra. The `*Run`
functions are executed in the following order:
* PersistentPreRun() 
* PreRun()
* Run()
* PostRun()
* PersistentPostRun()

To configure the `PreRun` for all commands we are going to use the following
functions:

```go
// addCommandWithConfigPreCheck adds a command to the parentCmd configuring a
// PreRunE function to ensure the configure command is executed before
// any other command
func AddCommandWithConfigPreCheck(parentCmd *cobra.Command, cmd *cobra.Command) {
	cmd.PreRunE = ConfigPreCheck
	parentCmd.AddCommand(cmd)
}

// configPreCheck verifies if the base configuration is set
func ConfigPreCheck(cmd *cobra.Command, args []string) error {
	validConfig := viper.InConfig(ClientIdFlag) &&
		viper.InConfig(ClientSecretFlag) &&
		viper.InConfig(APIEndpointFlag) &&
		viper.InConfig(TokenEndpointFlag)

	if !validConfig {
		return fmt.Errorf(
			"invalid CLI configuration: " +
				"please run `learning-go-api configure`")
	}

	return nil
}
```

When we implement our first command we'll see how to use all those functions.

## Unit testing

🏋️‍♀️ __CHALLENGE__: try to implement this by yourself before proceeding.

The contents of the `config_test.go` are:

```go
package config

import (
	"fmt"
	"math/rand"
	"os"
	"testing"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
	"github.com/stretchr/testify/assert"
)

func TestGetString(t *testing.T) {
	// arrange
	key := "someconfig"
	value := "somevalue"
	viper.Set(key, value)

	// act
	returnValue := GetString(key)

	// assert
	assert.Equal(t, value, returnValue)
}

func TestSet(t *testing.T) {
	// arrange
	key := "someconfig"
	value := "somevalue"

	// act
	Set(key, value)

	// assert
	returnValue := viper.Get(key)
	assert.Equal(t, value, returnValue)
}

func TestCreateConfigFile(t *testing.T) {
	// arrange
	home, _ := os.UserHomeDir()
	ext := "yaml"
	name := fmt.Sprintf(".learning-go-cli-test-%d", rand.Uint64())

	// act
	fileName, err := CreateConfigFile(home, name, ext)

	// assert
	if err != nil {
		assert.Fail(t, "error creating config file")
	}
	defer os.Remove(fileName)
	assert.Equal(t, fmt.Sprintf("%s/%s.%s", home, name, ext), fileName)
}

func TestConfigPreCheckReturnsErrorIfMissingConfigs(t *testing.T) {
	// arrange
	fileName := CreateFakeConfigFile(t)
	defer os.Remove(fileName)

	// act
	err := ConfigPreCheck(&cobra.Command{}, []string{})

	// assert
	print(err)
	assert.Error(t, err)
}

func TestConfigPreCheckReturnsNoErrorIfConfigsFound(t *testing.T) {
	// arrange
	fileName := CreateFakeConfigFile(t)
	defer os.Remove(fileName)
	viper.ReadInConfig()

	// act
	err := ConfigPreCheck(&cobra.Command{}, []string{})

	// assert
	assert.NoError(t, err)
}

func TestWriteAuthenticationConfig(t *testing.T) {
	// arrange
	fileName := CreateFakeConfigFile(t)
	defer os.Remove(fileName)

	apiEndpoint := "fake_endpoint_2"
	tokenEndpoint := "fake_endpoint_2"
	clientId := "fake_client_id_2"
	clientSecret := "fake_client_secret_2"

	// act
	WriteAuthenticationConfig(clientId,
		clientSecret,
		apiEndpoint,
		tokenEndpoint)

	// assert
	assert.Equal(t, GetString(ClientIdFlag), clientId)
	assert.Equal(t, GetString(ClientSecretFlag), clientSecret)
	assert.Equal(t, GetString(TokenEndpointFlag), tokenEndpoint)
	assert.Equal(t, GetString(APIEndpointFlag), apiEndpoint)
}

func TestInitConfig(t *testing.T) {
	// act
	InitConfig()

	// assert
	assert.NotEmpty(t, viper.ConfigFileUsed())
	assert.Contains(t, viper.ConfigFileUsed(), "learning-go-cli")
}

func TestAddCommandWithConfigPreCheck(t *testing.T) {
	// arrange
	cmd := &cobra.Command{}
	parentCmd := &cobra.Command{}

	// act
	AddCommandWithConfigPreCheck(parentCmd, cmd)

	// assert
	assert.NotNil(t, cmd.PreRunE)
	assert.Contains(t, parentCmd.Commands(), cmd)
}
```

# Next
 
The next section is
[CLI building blocks: authentication](it7-cli-building-blocks-authentication.md).