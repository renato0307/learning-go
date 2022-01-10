# Introduction to Cobra

To make our CLI simpler to implement and maintain, we are going to use Cobra to
help us build it.

From the [GitHub repository README](https://github.com/spf13/cobra):

> Cobra is both a library for creating powerful modern CLI applications as well
> as a program to generate applications and command files.
>
> Cobra is also an application that will generate your application scaffolding
> to rapidly develop a Cobra-based application.
> 
> Cobra is used in many Go projects such as Kubernetes, Hugo, and Github CLI 
> to name a few.

## Installation

Let's first install Cobra:

```sh
go install github.com/spf13/cobra/cobra
```

Check if Cobra is working by running the following in the command line:

```sh
cobra
go run main.go
```

## Initializing the CLI

The following command will create the barebones CLI project:

```sh
cobra init
```

It should return something like:

```
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.
```

The `main.go` file will be updated and also a `cmd` folder will be created with
the basic code.

The `main.go` file is really simple:

```go
package main

import "github.com/renato0307/learning-go-cli/cmd"

func main() {
	cmd.Execute()
}
```

Let's check and do some changes in the `root.go` file inside the `cmd` folder.

The first thing we need to do is change the `Short` and `Long` descriptions of
the root command:

```go
var rootCmd = &cobra.Command{
	Use:   "learning-go-cli",
	Short: "CLI for the learning-go-api"
	Long: `The learning-go-api provides utility functions like UUID generation,
a currency converter, a JWT debugger, etc.`,
}
```

If you run it again:

```sh
go run main.go
```

Now you'll see the following as output:

```
The learning-go-api provides with utility functions like UUID 
generation, a currency converter, etc.

```

Additionally clear all contents in the `init` function:

```go
func init() {
}
```

----
🕵️‍♀️ __GO-EXTRA: The init functions__

The `init` function in Go is used when we need to execute tasks before doing all
other things in that file. That means it is suited to do initializations.

The `init` function takes no arguments and has no returns.

Each of the Go source files in a package can have its own `init` function.

----

## Adding version support

To add version support we need to modify the root command initialization:

```go
var rootCmd = &cobra.Command{
	Use:   "learning-go-cli",
	Short: "CLI for the learning-go-api",
	Long: `The learning-go-api provides with utility functions like UUID
generation, a currency converter, a JWT debugger, etc.`,
	Version: "0.0.1",
}
```

With this change, we can now run:

```sh
go run main.go --version
```

And the result will be:

```
learning-go-cli version 0.0.1
```