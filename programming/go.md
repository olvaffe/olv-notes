Go
==

## Installation

- installation
  - download `https://golang.org/dl/go<ver>.linux-amd64.tar.gz`
  - untar it to `/usr/local/go`
  - add `/usr/local/go/bin` to `$PATH`
- `go env` prints the current environment
  - `GOROOT` is `/usr/local/go`
- the workspace is `$HOME/go`, or `$GOPATH`
  - packages are cloned to `src`
  - built packages are installed to `pkg` and are ready for importing
- install another go version as a package
  - `go get golang.org/dl/go<ver>`
  - this downloads `go<ver>` binary to `$GOPATH/bin`
  - running `go<ver>` prompts that the SDK is missing
    - `go<ver> download` downloads the tarball and untars it to
      `$HOME/sdk/go<ver>`
