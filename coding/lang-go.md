Go
==

## Installation

- installation
  - download `https://go.dev/dl/go<ver>.linux-amd64.tar.gz`
  - untar it to anywhere
  - add `$GOROOT/bin` to `$PATH`
- `go env` prints the current environment
  - `GO111MODULE` is empty
    - to run old go code, `GO111MODULE=off` might be necessary
  - `GOBIN` is empty, which defaults to `$GOPATH/bin`
  - `GOPATH` is `$HOME/go`, the workspace
    - packages are cloned to `$GOPATH/src`
    - built packages are installed to `$GOPATH/pkg` and are ready for
      importing
  - `GOROOT` is the top directory of the go installation
- install another go version as a package
  - `go install golang.org/dl/go<ver>@latest`
    - this downloads `go<ver>` binary to `$GOPATH/bin`
  - running `go<ver>` prompts that the SDK is missing
    - `go<ver> download` downloads the tarball and untars it to
      `$HOME/sdk/go<ver>`
- go standard library
  - `go install std` pre-compiles the standard library
  - some old build systems require the pre-compiled standard library to be
    installed to `$GOROOT/pkg`, like before 1.20
    - `GODEBUG=installgoroot=all go install std`
