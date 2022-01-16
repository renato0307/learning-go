# Start the library project

In this step we will:

1. Create the library folder
1. Init the git repository
1. Enable dependency tracking for the Go code
1. Create the first Go file
1. Create a repository in github.com
1. Commit and push the initial version

The following commands are needed for steps 1, 2, and 3:

```sh
mkdir learning-go-lib && cd learning-go-lib
git init .
go mod init github.com/renato0307/learning-go-lib
```

Replace the `renato0307` part by the correct value for you. This should be the
value of a GitHub repository where you will store the project.

The `go mod init` command initializes and writes a new `go.mod` file in the
current directory, in effect creating a new module rooted at the current
directory.

The `go.mod` file describes the module’s properties, including its dependencies
on other modules and on versions of Go.

Each time a new dependency is added running the `go mod tidy` command will
update the `go.mod` file.

The `go` command maintains a file named `go.sum` containing the expected 
cryptographic hashes of the content of specific module versions.


From the official
[blog post about Go Modules](https://go.dev/blog/using-go-modules):

> The go command uses the `go.sum` file to ensure that future downloads of these
> modules retrieve the same bits as the first download, to ensure the modules
> your project depends on do not change unexpectedly, whether for malicious,
> accidental, or other reasons. Both `go.mod` and `go.sum` should be checked
 into version control.

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

You should see in the output:

```terminal
Hello, World!
```

Next, go to [https://github.com]() and create a repository.

In my case I named it `learning-go-lib`.

After the repository is created, add it as remote origin and do the initial commit and push:

```sh
echo "# learning-go-lib" >> README.md
git add .
git remote add origin git@github.com:renato0307/learning-go-lib.git
git commit -m "initial version"
git branch -M main
git push -u origin main
````

With this all set, we are ready to start doing some real code.