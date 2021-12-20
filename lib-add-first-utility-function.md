# Adding the first utility function

As described in the [What are we going to build?](intro-what-are-we-going-to-build.md)
section, our utilities are going to be organized into categories, like Programming
or Finance.

As the number of categories and utilities will grow with time, a proper project
structure is needed to ensure we keep the code organized and easily maintainable.

The typical project layout of a Go project can be found
at https://github.com/golang-standards/project-layout.

I want to highlight the following parts:

>`/cmd`
> Main applications for this project. The directory name for each application 
> should match the name of the executable you want to have (e.g., /cmd/myapp).

> `/pkg`
> Library code that's ok to use by external applications (e.g., /pkg/mypubliclib).
> Other projects will import these libraries expecting them to work, so think twice
> before you put something here