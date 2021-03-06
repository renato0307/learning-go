
# Prepare the Development Environment

## Installing Go

Follow the instruction available at [https://go.dev/doc/install](https://go.dev/doc/install).

## Installing and configuring IDE

We are going to use Visual Studio Code (vscode).

Follow the installation instruction available at [https://code.visualstudio.com](https://code.visualstudio.com).

To use vscode for Go development is recommended to install the Go extension:
1. Go to Extensions
2. Search for "go" or "golang" and select the "Go" extension
3. Press install

![Installing the Go extension for Visual Studio Code](/assets/prepare-dev-env-1.png)

After, execute the "Go: Install/Update Tools" command using the Command Palette
(Ctrl+Shift+P on Linux/Windows or Cmd+Shift+P on Mac OS).

![Install Go tools](/assets/prepare-dev-env-2.png)

Choose the "Install All" option when prompted.

![Install Go tools prompt](/assets/prepare-dev-env-3.png)

The end result should be similar to:

![Install Go tools result](/assets/prepare-dev-env-4.png)

## Installing Docker

We will need Docker to build and run containers. 

Install it using the instructions available at
[https://docs.docker.com/engine/install](https://docs.docker.com/engine/install).

## Installing curl or httpie

We'll use a command line interface to do requests to our API. That can be
achieve with [curl](https://curl.se) or
[httpie](https://github.com/httpie/httpie).

I like the use the latest, httpie, because it tries to make CLI interaction
with web services as human-friendly as possible.

 # Next
 
 The next section is
 [Please do a Go tour](intro-go-tour.md).