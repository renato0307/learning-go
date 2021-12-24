# GitHub actions for the Library

The GitHub actions for the library are going to be pretty simple to start with.

We want to run the tests to ensure everything is OK, before each push or
pull request.

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

This workflow on any `push` or `pull_request` will run tests recursively.

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

| === RUN   TestNewUuidWithHyphen
| --- PASS: TestNewUuidWithHyphen (0.00s)
| === RUN   TestNewUuidWithoutHyphen
| --- PASS: TestNewUuidWithoutHyphen (0.00s)
| PASS
| ok    github.com/renato0307/learning-go-lib/programming       0.003s
[Test/build]   ✅  Success - Test
```

After just commit and push the workflow.

```sh
git add .
git commit -m "chore: add test github workflow"
git push
```

After a couple of minutes, head to GitHub and check the result:

Click on the green arrow and after in the "Details" link as illustrated below:

![Library](/assets/github-actions-for-the-library-1.png)

On the details you can expand the "Test" step and check for details:

![Library](/assets/github-actions-for-the-library-2.png)

