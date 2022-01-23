
# What are we going to build?

The goal is to build an API with utility functions.

![C4 Context](/assets/c4-model-context.png)

The system is to be used by a technical savvy user, via a REST endpoint or a
command line interface (CLI).

The system can use external APIs for specialized functionalities or data, for
example, to fetch exchange rates for currency conversions.

The system will be supported by two applications, the API and the CLI. The CLI
is just a thin wrapper calling the API. All the logic is inside the API.

![C4 Context](/assets/c4-model-containers.png)

The API is divided into two components, a API exposed via a web server and
a Library responsible by implementing the logic of the utility functions.

![C4 Context](/assets/c4-model-components.png)

Each component (library, API and CLI) are going to be implemented with
different projects.

__NOTE__: diagrams use [the C4 model](https://c4model.com/)
 for visualizing software architecture.

 # Next
 
 The next section is
 [Prepare the development environment](intro-prepare-dev-env.md).