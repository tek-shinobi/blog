---
title: "Golang:Go Tools Cheatsheet"
date: 2021-06-21T11:01:27+03:00
draft: false 
categories: ["golang"]
tags: ["tools", "automation"]
---
> __Note:__
>Starting in Go 1.17, installing executables with `go get` is deprecated. go install may be used instead. In Go 1.18, `go get` will no longer build packages...
>
>In other words, `go get` in 1.18 and beyond will no longer install executables. Use `go install`

Go has an impressive number of command line tools integreated into `go <tool_name>` that us developers use daily. You can get a list of them by
```shell
go help | bat -l man
```
> `bat` is similar to `cat` but supports some goodies syntax highlighting, ability to do git diff. Like in above case, am supplying the syntax formatting to be applied via `-l man` meaning `--language man`
>
> to install it, `sudo apt install bat`
>
> note that due to name collision with another package, the executable that is actually installed is called `batcat`. This is an apt issue. See here: https://github.com/sharkdp/bat#on-ubuntu-using-apt

Many of these tools are used internally by IDEs to give us linting, formatting on save etc but knowing these tools is essential as they serve a cruicial role in the CI pipeline, aka automation. Also, using them directly enables us to also use all the options exposed by these tools, something not possible when IDE is managing this for you.

Let's start with the most basic:

```shell
# will list all the env vars in the current proj directory.
# export to json format to pretty print via jq
go env --json | jq .
# or query a specific value from json output
go env --json | jq -r '.GOPROXY' 
# or in paginated mode using bat
go env | bat -l ini
```
You can also see specific env vars like so:
```shell
go env GOPROXY GOROOT GOCACHE
```

## gofmt and goimports
#### gofmt
```shell
# list files whose formatting differs from gofmt's
gofmt -l
# code format: simplify and write results to (source) file
gofmt -s -w
```

#### goimports
`goimports` is superset of `gofmt`. It does everything that `gofmt` does and also fixes imports (meaning add missing imports or remove redundant imports)
```shell
goimports -w
```
the `-w` flags writes the changes to the source file in place.

## go doc
`go doc` is a command to view the documentation. Its often integrated into IDEs. If you want to do it in terminal, 
```shell
# display all documentation for strings package
# pipe it to less for searching and pagination
go doc -all strings | less
# to view documentation for specific
go doc strings.Replace
```
if you want to see the source code as well, pass the `-src` flag
```shell
go doc -src strings.Replace | bat -l go
```
the above will show you the source code as well, directly in terminal

## godoc
You can use `godoc` (as a single word) to launch an local server and it will give you all the documentation + some books like effective Go, running on localhost. Very useful when you are offline and need documentation
```shell
# install godoc
go install golang.org/x/tools/cmd/godoc@latest
# now use godoc
godoc -http=:6060 &
# launch
open http://localhost:6060
# once done kill the godoc process to kill the server
pkill godoc
```

## Dependency Inspection
The main tool to do dependency inspection is `go list`. Dependencies being all the packages I am importing in different files.
```shell
# get full description of list doc
go help list | bat -l man
# to list all the packages my project is importing (direct and indirect)
go list all
# to list all the modules my project is importing (direct and indirect) along with their module version
go list -m all
# introspect Go packages and their interdependencies
go list  -f '{{ join .Imports "\n" }}'
# list all modules
go list -m all
# list outdated direct modules
go list -f '{{if and (not .Indirect) .Update}}{{.}}{{end}}' -u -m all

# or use depth for different visualization
# here I am asking why regexp is used in my project
# depth actually scans direct and indirect packages and builds a dependency tree
depth -explain regexp gtoken
```

For `list`, there is a flag `-f` that you can use to format the output but the same flag can also be used to filter the result. You can filter based on package structure or based on module structure (refer to `go help list | bat -l man` to see `Package` and `Module` structs used)

for example:
```shell
go list -f '{{ join .Imports "\n" }}'
```
here, we collect all direct package imports from all files, join them, split them by newline nad then display them on STDOUT. Also, the `{{ }}` style syntax is standard go template syntax (this template sytax is commonly used in places like Hugo markdown templates)

A more complex example: here we are listing all modules in our project (remember, the "-m" flag), that are direct imports and have updates available
```shell
go list -f '{{if and (not .Indirect) .Update}}{{.}}{{end}}' -u -m all
```
note here were are using `not .Indirect` because that is the flag for direct.

## Licensing tools
`golicense` To analyze the licenses used by third party libraries I am importing. This is v important since Go is statically built, and compiles to a single binary file. So the code from the 3rd party libraries is embedded in the single binary file. So its important to understand that the code is compliant to the licenses and if ther are some licenses to avoid, you should know this.
```shell
# .golicense.json is the configuration file for golicense. Its optional, but you will need it if you want to filter based on licenses, like disallow some license types
bat ../../.golicense.json
# scan for embedded licenses
golicense -verbose -license ../../.golicense.json .bin/gtoken
```
The thing to know is that golicense scans the binary file and not the source code for all the 3rd party packages and then downloads their license files from their import paths (like from their github repo).

example `.golicense.json` file:
```json
{
    "allow": ["MIT", "Apache-2.0", "MPL-2.0", "BSD-3-Clause", "ISC"],
    "deny": ["LGPL-2.0-or-later", "LGPL-3.0-or-later"],
    "override": {
        "github.com/russross/blackfriday": "BSD-2-Clause",
        "sigs.k8s.io/yaml": "MIT"
    }
}
```
The above `.golicense.json` file specifically lists the licenses to allow, the license I would like to deny.
`override` is used for packages that golicense fails to detect license of. Sometimes, licenses are not defined properly or in the standard way in packages, hence the detection issue. So, use `override` allows you to manually add the license type for those packages.

## Builds
Go can build for multiple architectures and OS (platforms).
If you want to see what architectures and platforms are supported:
```shell
go tool dist list
```

In order to compile binary to a specific platform and architecture, I need to provide two environment variables (GOOS and GOARCH):
```shell
GOOS=linux GOARCH=amd64 go build -o=bin/goapp_linux .
```
the above command will build an executable for linux on amd64.
```shell
GOOS=windows GOARCH=amd64 go build -o=bin/goapp_windows .
```
the above command will build an executable for windows on amd64.

to inspect the compiled binary file:
```shell
file bin/goapp_linux
```

So, its very easy to cross compile your application for multiple platforms.

#### build tags
if you want a go file to only be compiled for certain GOOS and GOARCH combination, put it as a comment at top (this is called build tag) in your go file
```go
// +build linux,arm64

package app
```
very useful if you have code that is OS specific or platform specific

#### linking
```shell
go tool lint -help
```
the above command will show multiple flags for linker. A very common flag is `-X` which, when used with `-ldflags` can used to override variable in your package (provide tha package name dot variable name) at build/link time. Very commonly used to embed the version info into the binary file, while building it.

For example:
```shell
export VERSION=$(git describe --tags --always --dirty)
export COMMIT=$(git rev-parse --short HEAD)
export BRANCH=$(git rev-parse --abbrev-ref HEAD)
export DATE=$(date +%FT%T%z)
# build binary
go build -ldflags "-X main.Version=${VERSION} -X main.GitCommit=${COMMIT} -X main.GitBranch=${BRANCH} -X main.BuildDate=${DATE}" -o bin/goapp
```
lets take git tag as a version.
`ldflags` stands for linker flags, and is used to pass in flags to the underlying linker

## Testing
Use `gotests` to generate table driven test templates. `mockery` to generate mock implementations for go interfaces.

```shell
gotests -all -w main.go
```
the above code will generate a `main_test.go` file and will autogenerate some table driven boilerplate. See here for details https://github.com/cweill/gotests

If you want a GoLand style colored test results, use `richgo` library https://github.com/kyoh86/richgo
you just use it like so `richgo test ./...` in place of `go test ./...`

## Static Analysis
`go vet` us used for static code analysis. Most IDEs have this built in, but this step is very useful in CI pipelines.
you can run it before build or after build. 

`errcheck` use this to search for unchecked errors

`golangci-lint` is an excellent tool to run multiple linters.
```shell
# run multiple linters with golangci-lint
golangci-lint run

# list available linters
golangci-lint linters

# run with all linters enabled
golangci-lint run --enable-all
```

## Makefile
example makefile:
```makefile
GOCMD=go
GOBUILD=$(GOCMD) build
GOCLEAN=$(GOCMD) clean
GOTEST=$(GOCMD) test
BINARY_NAME=app

all: test build
build:
        $(GOBUILD) -o $(BINARY_NAME) -v
test:
        $(GOTEST) -v ./...
clean:
        $(GOCLEAN)
        rm -f $(BINARY_NAME) $(BINARY_LINUX)
```

This article is sourced from this presentation: https://www.youtube.com/watch?v=DUnzcNEImsc&t=1516s

The presentation material used is here: https://github.com/alexei-led/presentations/tree/master/go-tools