The Cargo Book
==============

## Introduction

## Chapter 1. Getting Started

- 1.1. Installation
- 1.2. First Steps with Cargo

## Chapter 2. Cargo Guide

- 2.1. Why Cargo Exists
- 2.2. Creating a New Package
- 2.3. Working on an Existing Package
- 2.4. Dependencies
- 2.5. Package Layout
- 2.6. Cargo.toml vs Cargo.lock
- 2.7. Tests
- 2.8. Continuous Integration
- 2.9. Cargo Home
- 2.10. Build Cache

## Chapter 3. Cargo Reference

- 3.1. Specifying Dependencies
- 3.2. The Manifest Format
  - `[package]` defines a package
  - targets
    - `[lib]` customizes the library target
    - `[[bin]]` customizes a binary target
    - `[[example]]` customizes an example target
    - `[[test]]` customizes a test target
    - `[[bench]]` customizes a bench target
  - dependencies
    - `[dependencies]` specifies library dependencies
      - `time = "0.1.12"` means
        - `time` crate on `crates.io`
          - `https://crates.io/crates/time`
        - version range `[0.1.12, 0.2.0)`
      - it is the same as `time = { version = "0.1.12", registry = "crates-io" }`
        - `version` specifies the version
        - `registry` specifies the registry
        - `path` specifies a local path (rather than a registry)
        - `git` (and `branch`, `tag`, `rev`) specifies a git repo (rather than
          a registry)
    - `[dev-dependencies]` specifies deps for examples, tests, and benchmarks
    - `[build-dependencies]` specifies deps for build scripts
    - `[target]` specifies platform-specific deps
  - `[badges]` specifies status dadges to display on a registry
  - `[features]` specifies conditional compilation features
  - `[lints]` configure linters for this package
  - `[patch]` override dependencies
    - `[patch.crates-io] foo = { path = 'local_path' }` to use local version
      of `foo` crate
  - `[profile]` compiler settings and optimizations
  - `[workspace]` defines a workspace that consists of multiple packages
    - `resolver = "2"` specifies the v2 dep resolver
    - `members = ["path1", "path2"]` specifies the member packages
- 3.3. Workspaces
- 3.4. Features
- 3.5. Profiles
- 3.6. Configuration
- 3.7. Environment Variables
- 3.8. Build Scripts
- 3.9. Publishing on crates.io
- 3.10. Package ID Specifications
- 3.11. Source Replacement
- 3.12. External Tools
- 3.13. Registries
- 3.14. Dependency Resolution
- 3.15. SemVer Compatibility
- 3.16. Future incompat report
- 3.17. Reporting build timings
- 3.18. Unstable Features

## Chapter 4. Cargo Commands

- 4.1. General Commands
- 4.2. Build Commands
- 4.3. Manifest Commands
- 4.4. Package Commands
- 4.5. Publishing Commands

## Chapter 5. FAQ

## Chapter 6. Appendix: Glossary

## Chapter 7. Appendix: Git Authentication
