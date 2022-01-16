# Start the API project

In this step we will:

1. Create the API folder
1. Init the git repository
1. Enable dependency tracking for the Go code
1. Create the first Go file
1. Create a repository in github.com
1. Commit and push the initial version

The following commands are needed for steps 1, 2, and 3:

```sh
mkdir learning-go-api && cd learning-go-api
git init .
go mod init github.com/renato0307/learning-go-api
```

Replace the `renato0307` part by the correct value for you. This should be the
value of a GitHub repository where you will store the project.

Next, open vscode by typing:

```sh
code .
```

Create a file named `main.go` with the following contents:

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "Hello, welcome to the learning-go-api",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

We are going to use [Gin Tonic](https://github.com/gin-gonic/gin) as the
HTTP web framework to develop our API.

To run this code, go back to the command line and enter:

```sh
go run main.go
```

Visit "localhost:8080" on browser or use `httpie`.

```sh
http localhost:8080
```

You should see a result similar to:

```terminal
HTTP/1.1 200 OK
Content-Length: 51
Content-Type: application/json; charset=utf-8
Date: Tue, 21 Dec 2021 06:52:21 GMT

{
    "message": "Hello, welcome to the learning-go-api"
}
```


Next, go to [https://github.com]() and create a repository.

In my case I named it `learning-go-api`.

After the repository is created, add it as remote origin and do the initial
commit and push:

```sh
echo "# learning-go-api" >> README.md
git add .
git remote add origin git@github.com:renato0307/learning-go-api.git
git commit -m "initial version"
git branch -M main
git push -u origin main
````

With this all set, we are ready to start doing some real code.