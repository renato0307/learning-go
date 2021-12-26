# Running the API using a container

If you're new to containers please check 
["What is a container?"](https://azure.microsoft.com/en-us/overview/what-is-a-container/#overview).

Basically (from [docker.com](https://www.docker.com/resources/what-container)):
> A container is a standard unit of software that packages up code and all its
> dependencies so the application runs quickly and reliably from one computing
> environment to another. A Docker container image is a lightweight, standalone,
> executable package of software that includes everything needed to run an
> application: code, runtime, system tools, system libraries and settings.

To create a container we first need to build the container image. When using
Docker this is achieved by creating a `Dockerfile`, a simple text file that
consists of instructions to build Docker images.

Go to the `learning-go-api` folder and create an empty `Dockerfile`:

```sh
touch Dockerfile
```

The contents of this file are the following. The comments explain each
instruction.

```docker
## We specify the base image we need for our
## go application
FROM golang:1.17.5-alpine

## Install git
RUN apk add git

## We create an /app directory within our
## image that will hold our application source
## files
RUN mkdir /app

## We copy everything in the root directory
## into our /app directory
ADD . /app

## We specify that we now wish to execute 
## any further commands inside our /app
## directory
WORKDIR /app

## Add this go mod download command to pull in any dependencies
RUN go mod download

## We run go build to compile the binary
## executable of our Go program
RUN go build -o main .

## Set Gin mode to release
ENV GIN_MODE "release"

## Our start command which kicks off
## our newly created binary executable
CMD ["/app/main"]
```

As the `Dockerfile` contains the instructions, we need to build it.

To do that run the following command:

```sh
docker build . -t learning-go-api:latest
```

This will build an image using the Dockerfile from the current folder.
The `-t` argument tags the image with the `learning-go-api:latest` value.

The output of the command will be similar to:

```
(...)
Step 7/9 : RUN go build -o main .
 ---> Running in 0301d92f9256
Removing intermediate container 0301d92f9256
 ---> 42c10d2ba203
Step 8/9 : ENV GIN_MODE "release"
 ---> Running in 1efcb9ccc1b6
Removing intermediate container 1efcb9ccc1b6
 ---> d21db2252826
Step 9/9 : CMD ["/app/main"]
 ---> Running in 386214bdffa7
Removing intermediate container 386214bdffa7
 ---> e4c3fcece865
Successfully built e4c3fcece865
Successfully tagged learning-go-api:latest
```

To run the API using execute the following command:

```sh
docker run -p 81:8080 learning-go-api:latest
```

The `-p` defines the mapping between the host port and the container port.
This means when each time we do a request to `localhost:81`, it will forward the
requests to port `8080` in the container.

Let's try it:

```sh
http localhost:81
```

The result should be similar to:

```
HTTP/1.1 200 OK
Content-Length: 51
Content-Type: application/json; charset=utf-8
Date: Sun, 26 Dec 2021 09:26:36 GMT

{
    "message": "Hello, welcome to the learning-go-api"
}
```

Finish up by committing and pushing the changes to GitHub.