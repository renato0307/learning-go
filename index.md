# Welcome to Learning Go

Learn Go by building a REST API and a Command Line Interface (CLI).

We will also cover:
* Unit testing
* Authentication & Authorization using OAuth2
* CI/CD with Kubernetes, Helm and Flux

__Are you ready?__

## Introduction

1. [What are we going to build?](intro-what-are-we-going-to-build.md)
1. [Prepare the development environment](intro-prepare-dev-env.md)
1. [Please do a Go tour](intro-go-tour.md)
1. [What you will learn about Go (checklist)](intro-go-checklist.md)

## Iteration 1 - library and API skeleton

1. [Start the project for the Library](it1-lib-start-the-project.md)
1. [Add programming/uuid generator function the Library](it1-lib-add-first-utility-function.md)
1. [Start the project for the API](it1-api-start-the-project.md)
1. [Add programming/uuid generator function to the API](it1-api-add-first-utility-function.md)
1. [Unit tests in the API using mocks](it1-api-unit-tests-with-mocks.md)

## Iteration 2 - basic CI/CD
1. [GitHub actions running locally](it2-github-action-running-locally.md)
1. [GitHub actions for the Library](it2-github-actions-for-the-library.md)
1. [Running the API using a container](it2-run-api-using-container.md)
1. [GitHub actions for the API](it2-github-action-for-the-api.md)

## Iteration 3 - consolidate concepts by adding more features
1. [Add programming/jwtdebugger to the library and API](it3-add-programming-jwt-debugger.md)
1. [Add finance/currency-converter to the library](it3-add-finance-currency-converter-lib.md)
1. [Add finance/currency-converter to the API](it3-add-finance-currency-converter-api.md)

## Iteration 4 - deploying to kubernetes
1. [Create a local kubernetes cluster](it4-create-local-k8s.md)
1. [Create a Helm chart for the API](it4-create-helm-chart-api.md)
1. [Deploy the API using Flux](it4-deploy-api-using-fluxcd.md)

## Iteration 5 - code improvements
1. [Change API file structure](it5-change-api-file-structure.md)
1. [Improve API error handling](it5-improve-api-error-handling.md)
1. [Add structured logs to the API](it5-add-logs-api.md)
1. [Make Gin use structured logs](it5-gin-structured-logging.md)

## Iteration 6 - authentication & authorization
1. [Configure Amazon Cognito as an OAuth2 server](it6-create-cognito-user-pool.md)
1. [Add authentication to the API](it6-add-authentication-api.md)
1. [Add authorization to the API](it6-add-authorization-api.md)

## Iteration 7 - implementing the CLI
1. [Start the project](it7-cli-start-the-project.md)
1. [Introduction to Cobra](it7-cli-cobra-intro.md)
1. [CLI building blocks: configuration](it7-cli-building-blocks-configuration.md)
1. [CLI building blocks: authentication](it7-cli-building-blocks-authentication.md)
1. [CLI building blocks: IO streams & testing helpers](it7-cli-building-blocks-iostreams.md)
1. [Add the configure command to the CLI](it7-cli-add-configure-cmd.md)
1. [Add the programming/uuid command to the CLI](it7-cli-add-programming-uuid-cmd.md)
1. [CI/CD for the CLI](it7-cli-ci-cd.md)
1. [Challenge: Add the programming/jwtdebugger command to the CLI](it7-cli-add-programming-jwtdebugger-cmd.md)
1. [Challenge: Add the finance/currency-converter command to the CLI](it7-cli-add-finance-currconv-cmd.md)

## Iteration 8 - increasing test coverage

1. How to check the test coverage?
1. Increase coverage in the Lib
1. Increase coverage in the API
1. Increase coverage in the CLI

## Conclusion

1. What's next?