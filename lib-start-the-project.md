# Start the library project

In this step we will:

1. Create the project folder
1. Create the library folder
1. Init the git repository
1. Enable dependency tracking for the Go code

The following commands are needed:

```bash
mkdir learning-go && cd learning-go
mkdir learning-go-lib && cd learning-go-lib
git init .
go mod init github.com/renato0307/learning-go.lib
```

Replace the `renato0307` part by the correct value for you.

Wheres a cast showing it:

[![asciicast](https://asciinema.org/a/oQn23v4QBRUgSbgOaJZJEtLWJ.svg)](https://asciinema.org/a/oQn23v4QBRUgSbgOaJZJEtLWJ)