# GitHub actions for the API

The GitHub actions for the API are going to be a little more complex than
the ones we did for the Library.

In this case we want:

1. To run the tests to ensure everything is OK, before each push or
pull request
1. Generate a new container image each time a new version of the API is created

## Running tests action

This is going to be very similar to the one we did for the Library. You can just
copy the same workflow file definition.

First create the workflows folder:

```sh
mkdir -p .github/workflows
```

Next create the `.github/workflows/test.yaml` file with the following contents:

```yaml
name: Test

on:
  push:
  pull_request:

jobs:

  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2

    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.17

    - name: Test
      run: go test -v ./...
```

This workflow will run tests recursively on any `push` or `pull_request` .

We can test this workflow using [act](it2-github-action-running-locally.md).

To execute `act` simulating a `push` event run the following command:

```sh
act push
```

The first execution will take some time as the docker containers need to be
downloaded but in the end you should see something like:

```
[Test/build] ⭐  Run Test

(...)

| PASS
| ok    github.com/renato0307/learning-go-api/programming       0.006s
[Test/build]   ✅  Success - Test
```

Finish up by committing and pushing the changes to GitHub.

After a couple of minutes, head to GitHub and check the result, like we did
for the library.


## Create container image action