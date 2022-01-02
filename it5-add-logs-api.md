# Add logs to the API

One of the problems of the current API implementation is the lack of logs,
besides the ones already added out-of-the-box by Gin.

Logs are one of the most valuable assets a developer can have for
troubleshooting purposes.

As we are adding logs from scratch, we are going to use `structured logging`,
which brings more structure and details to our logs.

The standard logging library does not support structured logging by default,
so we are going to use one of the external loggers supporting it.

Our pick is [rs/zerolog](https://github.com/rs/zerolog).

Let's start by a simple example.

