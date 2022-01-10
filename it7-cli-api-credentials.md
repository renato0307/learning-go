# Configuration of API credentials

All commands will require API authentication. To avoid having to enter the
credentials all the time we will make a command just to store the API
authentication information in a file so all commands can use them.

Create the `configure.go`  file in the `cmd` folder.

The contents of this file are:

```go
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
)

// configureCmd represents the configure command
var configureCmd = &cobra.Command{
	Use:   "configure",
	Short: "Configures the CLI",
	Long:  `Allows to define the API endpoints and the client credentials`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("configure called")
	},
}

func init() {
	rootCmd.AddCommand(configureCmd)
}
```