# Start the library project

In this step we will:

1. Create the project folder
1. Create the library folder
1. Init the git repository
1. Enable dependency tracking for the Go code
1. Create the first Go file
1. Create a repository in github.com
1. Commit and push the initial version

The following commands are needed for step 1, 2, 3 and 4:

```sh
mkdir learning-go && cd learning-go
mkdir learning-go-lib && cd learning-go-lib
git init .
go mod init github.com/renato0307/learning-go.lib
```

Replace the `renato0307` part by the correct value for you. This should be the
value of a GitHub repository where we will store the project.

Next, open vscode by typing:

```sh
code .
```

Create a file named `main.go` with the following contents:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

To run this code, go back to the command line and enter:

```sh
go run main.go
```

You must see in the output:

```
Hello, World!
```

With this all set, we are ready to start.