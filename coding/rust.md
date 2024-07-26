Rust
====

## Resources

- <https://github.com/rust-lang>
  - <https://github.com/rust-lang/rust> has rustc, std, and docs
    - <https://github.com/rust-lang/backtrace-rs> is `std::backtrace`
    - <https://github.com/rust-lang/stdarch> is `std::arch`
    - <https://github.com/rust-lang/book>
    - <https://github.com/rust-lang/edition-guide>
    - <https://github.com/rust-embedded/book>
    - <https://github.com/rust-lang/nomicon> is advanced/unsafe rust
    - <https://github.com/rust-lang/reference>
    - <https://github.com/rust-lang/rust-by-example>
    - <https://github.com/rust-lang/rust/tree/master/src/doc/style-guide>
    - <https://github.com/rust-lang/rust/tree/master/src/doc/unstable-book>
  - binaries
    - <https://github.com/rust-lang/cargo>
    - <https://github.com/rust-lang/rust-bindgen>
    - <https://github.com/rust-lang/rust-clippy>
    - <https://github.com/rust-lang/rustfmt>
    - <https://github.com/rust-lang/rustup>
  - libraries
    - <https://github.com/rust-lang/cfg-if>
    - <https://github.com/rust-lang/flate2-rs>
    - <https://github.com/rust-lang/futures-rs>
    - <https://github.com/rust-lang/getopts>
    - <https://github.com/rust-lang/glob>
    - <https://github.com/rust-lang/libc>
    - <https://github.com/rust-lang/log>
    - <https://github.com/rust-lang/regex>
    - <https://github.com/rust-lang/socket2>
  - docs
    - <https://github.com/rust-lang/api-guidelines>
    - <https://github.com/rust-lang/async-book>
    - <https://github.com/rust-lang/rfcs>
- <https://github.com/rust-unofficial>
  - <https://github.com/rust-unofficial/awesome-rust>
  - <https://github.com/rust-unofficial/patterns>
- popular crates
  - <https://crates.io/crates/anyhow>
  - <https://crates.io/crates/bitflags>
  - <https://crates.io/crates/bytes>
  - <https://crates.io/crates/cfg-if>
  - <https://crates.io/crates/clap>
  - <https://crates.io/crates/either>
  - <https://crates.io/crates/itertools>
  - <https://crates.io/crates/libc>
  - <https://crates.io/crates/log>
  - <https://crates.io/crates/memoffset>
  - <https://crates.io/crates/once_cell>
  - <https://crates.io/crates/proc-macro2>
  - <https://crates.io/crates/quote>
  - <https://crates.io/crates/regex>
  - <https://crates.io/crates/scopeguard>
  - <https://crates.io/crates/serde>
  - <https://crates.io/crates/smallvec>
  - <https://crates.io/crates/syn>
  - <https://crates.io/crates/thiserror>

## Installation

- `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
  - this downloads and runs `rustup-init.sh`, which downloads and runs
    <https://static.rust-lang.org/rustup/dist/x86_64-unknown-linux-gnu/rustup-init>
- `rustup-init`
  - copies itself as `rustup` to `~/.cargo/bin`, or `$CARGO_HOME/bin`
    - yes, `rustup-init` is installed as `rustup` itself
  - creates hardlinks such as `cargo`, `rustc`, etc under `$CARGO_HOME/bin`
    - yes, toolchain binaries are hardlinks to `rustup`
    - when executed, they find the binary of the same name under
      `$RUSTUP_HOME/toolchains/stable-x86_64-unknown-linux-gnu/bin` and execve
  - runs `rustup` to install these components to `~/.rustup`, or `$RUSTUP_HOME`
    - cargo
    - clippy
    - rust-docs
    - rust-std
    - rustc
    - rustfmt
  - edit `.profile` to source `$CARGO_HOME/env`
    - `$CARGO_HOME/env` adds `$CARGO_HOME/bin` to `$PATH`

## Cross-Compiler

- <https://rust-lang.github.io/rustup/cross-compilation.html>
- `rustup target list` for supported targets
- `rustup target add <triple>` to add the toolchain
- `cargo build --target <triple>` to build
- a linker must be specified separately
  - edit `~/.cargo/config.toml` to add

    [target.<triple>]
    linker = "..."
  - or, set the `CARGO_TARGET_<triple>_LINKER` envvar
- aarch64
  - `rustup target add aarch64-unknown-linux-gnu`
  - `cargo build --target aarch64-unknown-linux-gnu`
  - install the linker separately and edit `~/.cargo/config.toml`

    [target.aarch64-unknown-linux-gnu]
    linker = "aarch64-linux-gnu-gcc"
- android
  - `rustup target add aarch64-linux-android`
  - `cargo build --target aarch64-linux-android`
  - install the ndk and edit `~/.cargo/config.toml`

    [target.aarch64-unknown-linux-gnu]
    linker = "<ndk>/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android34-clang"

## cargo

- `~/.cargo` or `$CARGO_HOME`
- `$CARGO_HOME/registry`
  - `index` appears to be
    `git clone --bare https://github.com/rust-lang/crates.io-index.git`
  - `cache` contains downloaded `.crate` files (can be renamed to `.tar.gz`)
  - `src` contains un-tared crates
- `cargo new <dir>` creates a new project
  - `src/main.rs` is the source code
  - `Cargo.toml` is the manifest
- `cargo build`
  - downloads dependencies to `$CARGO_HOME/registry/src`
  - builds dependencies in `target/debug/deps`
  - `cargo build --examples` builds binary crates under `src/examples`
- `cargo test`
  - it builds and runs tests
  - unit tests go to `src/`
  - integration tests go to `tests/`
- `cargo bench`
  - it builds and runs benches
  - unit benches go to `src/`
  - integration benches go to `benches/`
- `Cargo.toml`
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

## The Book - Chapter 1. Getting Started

- 1.1. Installation
- 1.2. Hello, World!
- 1.3. Hello, Cargo!

## The Book - Chapter 2. Programming a Guessing Game

## The Book - Chapter 3. Common Programming Concepts

- 3.1. Variables and Mutability
- 3.2. Data Types
- 3.3. Functions
- 3.4. Comments
- 3.5. Control Flow

## The Book - Chapter 4. Understanding Ownership

- 4.1. What is Ownership?
- 4.2. References and Borrowing
- 4.3. The Slice Type

## The Book - Chapter 5. Using Structs to Structure Related Data

- 5.1. Defining and Instantiating Structs
  - `struct Foo {...}` defines a struct
- 5.2. An Example Program Using Structs
- 5.3. Method Syntax
  - `impl Foo {...}` defines associated funcs for `Foo`
  - an associated func is a method only when its first param is some variant
    of `self`
    - `&self` is replaced by `self: &Self`
    - `Self` is replaced by `Foo`, because we are in `impl Foo` block
  - e.g., in `let mut v = Vec::new(); v.push(1);`,
    - `new` is not a method
    - `push` is a method

## The Book - Chapter 6. Enums and Pattern Matching

- 6.1. Defining an Enum
  - `Option` enum
    - enum vals are `None` and `Some(T)`
- 6.2. The match Control Flow Construct
- 6.3. Concise Control Flow with if let

## The Book - Chapter 7. Managing Growing Projects with Packages, Crates, and Modules

- rust module system
  - package: a cargo feature to build, test, and share crates
  - crates: a tree of modules that produce a library or executable
  - modules and uses: code organization and scoping
  - paths: a way of naming items such as structs, functions, and modules
- 7.1. Packages and Crates
  - a package is a bundle of crates
  - `Cargo.toml` describes how to build the crates
  - there is at most one library crate
    - by convention, the crate root is `src/lib.rs`
  - there can many binary crates
    - by convention, the crate root is `src/main.rs`
    - each file under `src/bin/` is a separate binary crate
- 7.2. Defining Modules to Control Scope and Privacy
  - modules are used to organize and scope code
  - code within a module is private by default
  - single-file module tree
    - `src/lib.rs` or `src/main.rs` is itself a module named `crate`
    - `mod foo { mod bar {...} ... }` in the file defines
      - module `crate::foo`
      - module `crate::foo::bar`
  - multi-file module tree
    - cargo only builds `src/lib.rs` or `src/main.rs`
    - `pub mod foo;` in the file tells rustc to build `src/foo.rs`
    - `pub mod bar;` in `src/foo.rs` tells rustc to build `src/foo/bar.rs`
  - there are also `pub(crate)`, `pub(super)`, etc., to export an item to the
    current crate or parent crate
- 7.3. Paths for Referring to an Item in the Module Tree
  - `pub` marks a `mod`/`fn`/`struct`/`enum` public
  - `crate::` refers to the root module
  - `super::` refers to the parent module
- 7.4. Bringing Paths Into Scope with the use Keyword
  - `use` brings a name into scope
    - `use ... as ...` to pick an alternative name
  - idiomatically,
    - if `crate::foo::bar` is a fn,
      - `use crate::foo;` and refer to it as `foo::bar`
    - if `crate::foo::Baz` is a struct,
      - `use crate::foo::Baz;` and refer to it as `Baz`
  - `pub use ...` makes a name public
  - external packages
    - if `Cargo.toml` lists external package `foo` as a dependency,
      - `foo` refers to the root module
  - `use std::io::{self, Write};` expands to
    - `use std::io;`
    - `use std::io::Write;`
  - `use foo::*` brings all public items under `foo` into scope
- 7.5. Separating Modules into Different Files
  - we can build a module tree in a single file
  - we can also build a module tree using multiple files and directories
  - cargo knows `src/main.rs` or `src/lib.rs` is the crate root
    - `mod foo` in the crate root tells cargo to build `src/foo.rs`
      - or the legacy `src/foo/mod.rs`
    - `mod bar` in `src/foo.rs` tells cargo to build `src/foo/bar.rs`
      - or the legacy `src/foo/bar/mod.rs`

## The Book - Chapter 8. Common Collections

- 8.1. Storing Lists of Values with Vectors
- 8.2. Storing UTF-8 Encoded Text with Strings
  - ways to convert `&str` to `String`
    - `"test".to_string()`, because `str` implements the
      `std::string::ToString` trait
      - the trait is automatically implemented for any type that implements
        the `Display` trait
    - `"test".to_owned()`, because `str` implements the `std::borrow::ToOwned`
      trait
      - `str` implemenets `ToOwned` with
        `String::from_utf8_unchecked(self.as_bytes().to_owned())`
    - `String::from("test")` because `String` implements `From<&str>`
      - it implements `From<&str>` with `s.to_owned()`
- 8.3. Storing Keys with Associated Values in Hash Maps

## The Book - Chapter 9. Error Handling

- 9.1. Unrecoverable Errors with `panic!`
- 9.2. Recoverable Errors with `Result`
  - `Result` enum
    - enum vals are `Ok(T)` and `Err(E)`
  - panic on errors
    - `unwrap` and `expect`
  - early return on errors
    - `?` operator
      - if `Ok(T)`, unwraps and returns `T`
      - if `Err(E)`, early returns `Err(E)`
        - it actually early returns `Err(From::from(e))`, to cast the error
          type when necessary
      - it works with `Option` as well
  - `std::error::Error` trait
    - while there is no restriction on `Err(E)`, it is generally a good idea
      for `E` to implement the `Error` trait
    - the `Error` trait requires `Debug` and `Display` traits
      - `pub trait Error: Debug + Display`
      - that is, a type implementing `Error` must also implement `Debug` and
        `Display`
    - the `Error` trait has these methods
      - `description` is deprecated because the `Display` trait req serves the
        same purpose
      - `cause` is deprecated by `source`
      - `source` returns `Option<&dyn Error>`, which is an optional low-level
        error wrapped in this high-level error (that is, the root source of
        this error)
      - `provide` returns associated payload (often backtrace) with this error
      - iow, when a user chooses to deal with the generic `Error` trait rather
        than each concrete error type, the trait provides
        - `Display` trait to return a string that describes this error
        - `source` method to return the root cause, if any
        - `provide` method to return additional info, if any
  - `std::io::Error` implements `std::error::Error` trait
    - `io::Error::new` creates an error from `ErrorKind` and a payload
      - `std::io::ErrorKind` is a plan enum
      - the payload can be many things, such as a string or another error
    - `io::Error::from_raw_os_error` creates an error from
      `std::io::RawOsError`, which is the os errno
  - `std::io::Result` is an alias of `std::result::Result<T, std::io::Error>`
  - best practices
    - our code generates `N` kinds of errors
    - our dependencies generate `M` error types
    - if we want to faithfully return the errors to the user, define a custom
      error type
      - an enum with `N+M` vals, with the `M` vals wrapping the `M` error
        types
      - impl `std::error::Error` to allow the user to handle the custom error
        generically
      - impl `From<T>` to allow the user (and ourselves) to propagate from
        `Err(T)` to our custom error type using `?`
      - this is a common case for libraries and `thiserror` crate is designed
        for the case
    - if we don't expect users to care and we just want to log the errors
      somewhere, define a trait object for `std::error::Error`
      - the trait object gives us a string representation of the error (and
        a bit more)
      - this is a common case for executables and `anyhow` create is designed
        for the case
- 9.3. To `panic!` or Not to `panic!`

## The Book - Chapter 10. Generic Types, Traits, and Lifetimes

- 10.1. Generic Data Types
  - turbofish syntax
    - when we have `struct Foo<T>`, we use `Foo<i32>` to specify a concrete
      type
    - when we have `fn foo<T>`, we use `foo::<i32>` instead to specify a
      concrete type
      - `::<>` is nicknamed turbofish
    - when `struct Foo<T>` has method `foo`, we use `Foo::<i32>::foo`
- 10.2. Traits: Defining Shared Behavior
  - `trait Foo { fn bar(&self)... }` defines trait `Foo`
  - `impl Foo for Baz` implements trait `Foo` for type `Baz`
    - trait `Foo` can also have non-methods such as `fn blah()`
      - that is, no `self`
      - `Baz::blah` invokes the func
      - `Foo::blah` also invokes the func if the type can be inferred
      - e.g.,
        - `let a : i32 = Default::default();`
        - `let b = i32::default();`
        - `let c = <i32 as Default>::default();`
  - trait bound
    - `fn some_func(foo: &impl Foo)` takes any type that implements
      `Foo` trait
    - it is a syntax sugar for `fn some_func<T: Foo>(foo: &T)`
      - or the newer `fn some_func<T>(foo: &T) where T: Foo`
    - multiple trait bounds are possible with `+`
      - `fn some_func(foo: &(impl Foo + Bar))`
      - `fn some_func<T: Foo + Bar>(foo: &T)`
    - `Sized` trait is special
      - if a type has a fixed-size at compile-time (not DST), the compiler
        implements `Sized` for the type automatically
      - `T` is assumed to have `Sized` bound (except for `Self` in traits)
      - `T: !Sized` means no `Sized` bound
      - `T: ?Sized` means either `Sized` or not
  - `trait Foo: Bar` means a type must implement `Bar` first to implement `Foo`
  - trait coherence
    - the goal is to make sure the compiler can always unambiguously resolve a
      trait func call to an impl
    - the overlay rule
      - if a type implements two traits, there must be no overlap between the
        two traits
    - the orphan rule
      - if a type implements a trait, at least one of them must be defined by
        the current crate
- 10.3. Validating References with Lifetimes

## The Book - Chapter 11. Writing Automated Tests

- 11.1. How to Write Tests
  - `cargo test` buids a test runner to run functions with `#[test]` attr
- 11.2. Controlling How Tests Are Run
  - `cargo test --help` for help
  - `cargo test -- --help` for test harness help
    - `--nocapture` or `--show-output` to show output
- 11.3. Test Organization
  - unit tests are added to modules directly
    - in `mod tests` submodules decorated with `#[cfg(test)]`
  - integration tests are added to `tests/`

## The Book - Chapter 12. An I/O Project: Building a Command Line Program

- 12.1. Accepting Command Line Arguments
- 12.2. Reading a File
- 12.3. Refactoring to Improve Modularity and Error Handling
- 12.4. Developing the Library’s Functionality with Test Driven Development
- 12.5. Working with Environment Variables
- 12.6. Writing Error Messages to Standard Error Instead of Standard Output

## The Book - Chapter 13. Functional Language Features: Iterators and Closures

- 13.1. Closures: Anonymous Functions that Capture Their Environment
- 13.2. Processing a Series of Items with Iterators
- 13.3. Improving Our I/O Project
- 13.4. Comparing Performance: Loops vs. Iterators

## The Book - Chapter 14. More about Cargo and Crates.io

- 14.1. Customizing Builds with Release Profiles
- 14.2. Publishing a Crate to Crates.io
- 14.3. Cargo Workspaces
- 14.4. Installing Binaries from Crates.io with cargo install
- 14.5. Extending Cargo with Custom Commands

## The Book - Chapter 15. Smart Pointers

- 15.1. Using `Box<T>` to Point to Data on the Heap
  - `let a = 5` and `let b = Box::new(5)`, one is on stack and the other is on
    heap
  - `Box<T>` is commonly used when
    - `T` is a recursive type, because rustc needs to know the type size at
      compile time
    - `T` is huge and we want to make sure it is never copied
    - `T` is a trait object
  - `Box<T>` implements `Deref` and `Drop` traits
- 15.2. Treating Smart Pointers Like Regular References with the `Deref` Trait
  - with `let x = 5;` and `let y = &x;`, `y` is a reference to `x` and must be
    dereferenced to get the value, such as in `*y == x`
  - if we have `let y = Box::new(x);` instead, we can still do `*y == x`
    because `Box` implements `Deref` trait
    - it implements the `deref` method which takes `&Box<T>` and returns `&T`.
    - that allows the compler to replace `*y` by `*(y.deref())`
  - deref coercion
    - it is for convenience
    - when a function takes `&str`, we can pass `&Box<String>` to it thanks to
      deref coercion
    - `&Box<String>` becomes `&String` which becomes `&str` through a series
      of `deref()`
  - there is also `DerefMut` trait for `&mut`
  - `Deref` and `AsRef`
    - when `T` wraps `U`, implementing `Deref` for `T` enables type coercion
      (implicit type conversion) to `U`
      - this is very convenient, and the key consideration is if the implicit
        conversion can be an unexpected surprise or not
      - this is done for `Box<T>` to `T`, `String` to `str`, etc.
    - when `T` casts to `U` at little or no cost, implementing `AsRef<U>` for
      `T` enables generic functions
      - e.g., if I have a function that takes a path, the param can be generic
        with trait bound `AsRef<Path>` to accept a wide range of types, such
        as `str`, `String`, `OsStr`, `OsString`, `Path`, `PathBuf`, etc.
- 15.3. Running Code on Cleanup with the `Drop` Trait
  - `Box` implements the `Drop` trait
    - when it goes out of scope, `drop` method is called automatically to free
      the storage
  - to drop early, `std::mem::drop` can be used
    - the prelude has `use std::mem::drop;` so it is just `drop`
- 15.4. `Rc<T>`, the Reference Counted Smart Pointer
- 15.5. `RefCell<T>` and the Interior Mutability Pattern
- 15.6. Reference Cycles Can Leak Memory

## The Book - Chapter 16. Fearless Concurrency

- 16.1. Using Threads to Run Code Simultaneously
- 16.2. Using Message Passing to Transfer Data Between Threads
- 16.3. Shared-State Concurrency
- 16.4. Extensible Concurrency with the Sync and Send Traits

## The Book - Chapter 17. Object Oriented Programming Features of Rust

- 17.1. Characteristics of Object-Oriented Languages
- 17.2. Using Trait Objects That Allow for Values of Different Types
  - given `trait Foo {}` and `impl Foo for Bar`
    - `Bar` implements the `Foo` trait
    - when a method is called, it can be resolved statically and no vtable is
      needed
  - on the other hand, `Box<dyn Foo>` is a trait object
    - `dyn` makes sure there is a vtable
    - when a method is called, it is resolved dynamitcally via the vtable
    - note that `Box<dyn Foo>` is a fat pointer
      - `std::mem::size_of::<Box<dyn Foo>>()` returns 16
      - there is 1 pointer to the concrete type and 1 pointer to the vtable
- 17.3. Implementing an Object-Oriented Design Pattern

## The Book - Chapter 18. Patterns and Matching

- 18.1. All the Places Patterns Can Be Used
- 18.2. Refutability: Whether a Pattern Might Fail to Match
- 18.3. Pattern Syntax

## The Book - Chapter 19. Advanced Features

- 19.1. Unsafe Rust
- 19.2. Advanced Traits
- 19.3. Advanced Types
- 19.4. Advanced Functions and Closures
- 19.5. Macros

## The Book - Chapter 20. Final Project: Building a Multithreaded Web Server

- 20.1. Building a Single-Threaded Web Server
- 20.2. Turning Our Single-Threaded Server into a Multithreaded Server
- 20.3. Graceful Shutdown and Cleanup

## The Book - Chapter 21. Appendix

- 21.1. A - Keywords
- 21.2. B - Operators and Symbols
- 21.3. C - Derivable Traits
  - `#[derive]` attr on a struct or an enum generates a default implementation
    for the specified trait
  - `#[derive(Debug)]` enables string formatting with `{:?}`
  - `#[derive(PartialEq)]` and `#[derive(Eq)]` enables `==` and `!=`
    - `Eq` trait cannot be implemented on floats, because `NaN != NaN`
  - `#[derive(PartialOrd)]` and `#[derive(Ord)]` enables `<`, `>`, `<=`, and
    `>=`
    - `Eq` trait cannot be implemented on floats, because `NaN` has no
      ordering
  - `#[derive(Clone)]` and `#[derive(Copy)]`
    - `Copy` trait is fast because it copies bits on stack
    - `Clone` trait can be slow becaise it might copies bits on heap
  - `#[derive(Hash)]` eanbles hashing
  - `#[derive(Default)]` enables default value
- 21.4. D - Useful Development Tools
- 21.5. E - Editions
- 21.6. F - Translations of the Book
- 21.7. G - How Rust is Made and “Nightly Rust”

## The Reference - Chapter 2. Lexical structure

- 2.1. Input format
- 2.2. Keywords
- 2.3. Identifiers
- 2.4. Comments
- 2.5. Whitespace
- 2.6. Tokens

## The Reference - Chapter 3. Macros

- 3.1. Macros By Example
- 3.2. Procedural Macros

## The Reference - Chapter 4. Crates and source files

- each compilation processes a single crate in the source form and produces
  the crate in the binary form (executable or library)
- a crate is a unit of compilation and linking
  - it contains a tree of modules
  - the root module is anonymous
- the compiler is always invoked with a single source file
  - the source file is the root module of a crate
  - it can reference external source files
  - each file is a module
    - but not every module has its own file
- each source file contains zero or more items
  - it may optionally begin with attrs
- a crate that contains a `main` function can be compiled to an executable
  - the root module must define `main`, or have `use foo::bar as main`
  - the main function must
    - have no args
    - have no trait/lifetime bounds
    - have no where clauses
    - have return type that implements `std::process::Termination`
      - such as `()`, `!`, `Infallible`, `ExitCode`
      - also `Result<T, E> where T: Termination, E: Debug`

## The Reference - Chapter 5. Conditional compilation

- source code can be conditionally compiled using
  - `cfg` attr
  - `cfg_attr` attr
  - `cfg` macro
  - each form takes a predicate that evaludates to true or false
  - a predicate is one of
    - a config option
      - a name or a key-val pair
    - `all(opt1,...)`
    - `any(opt1,...)`
    - `not(opt)`
- config options
  - arbitrary options
    - `rustc --cfg foo` sets `foo`
    - cargo specifies `--cfg feature` for example
  - target options
    - `target_arch`, with values such as
      - `x86`, `x86_64` `arm`, `aarch64`, etc.
    - `target_feature`, with values such as
      - `avx`, `avx2`, `rdrand`, etc.
    - `target_os`, with values such as
      - `windows`, `macos`, `ios`, `linux`, `android`, `freebsd`, etc.
      - `none` for embedded targets
    - `target_family`, with values such as
      - `unix`, `windows`, `wasm`, etc.
    - `target_env`, with values such as
      - usually empty, but can be `gnu`, `msvc`, `musl`, etc.
    - `target_abi`, with values such as
      - usually empty, but can be `llvm`, etc.
    - `target_endian`, with values `little` or `big`
    - `target_pointer_width`, with values `16`, `32`, or `64`
    - `target_vendor`, with values such as
      - `apple`, `pc`, `unknown`, etc.
    - `target_has_atomic`, with values such as
      - `8`, `16`, `34`, `64`, `128`, `ptr`, etc.
  - when `target_family` is `unix` or `windows`, shorthands are set as well
    - `unix` if unix
    - `windows` if windows
  - other options
    - `test`, if rustc is invoked with `--test`
    - `debug_assertions`, if compiling without optimization
    - `proc_macro`, if the crate type is `proc_macro`
    - `panic`, with values such as
      - `abort`, `unwind`, etc.

## The Reference - Chapter 6. Items

- 6.1. Modules
- 6.2. Extern crates
- 6.3. Use declarations
- 6.4. Functions
- 6.5. Type aliases
- 6.6. Structs
- 6.7. Enumerations
- 6.8. Unions
- 6.9. Constant items
- 6.10. Static items
- 6.11. Traits
- 6.12. Implementations
- 6.13. External blocks
- 6.14. Generic parameters
- 6.15. Associated Items

## The Reference - Chapter 7. Attributes

- syntax
  - `#![attr]` applies to the item that the attr declared within
    - when declared inside a module, it applies to the module
  - `#[attr]` applies to the item that follows
- built-in attributes
  - Conditional compilation
    - `cfg` Controls conditional compilation.
    - `cfg_attr` Conditionally includes attributes.
  - Testing
    - `test` Marks a function as a test.
    - `ignore` Disables a test function.
    - `should_panic` Indicates a test should generate a panic.
  - Derive
    - `derive` Automatic trait implementations.
    - `automatically_derived` Marker for implementations created by derive.
  - Macros
    - `macro_export` Exports a macro_rules macro for cross-crate usage.
    - `macro_use` Expands macro visibility, or imports macros from other
      crates.
    - `proc_macro` Defines a function-like macro.
    - `proc_macro_derive` Defines a derive macro.
    - `proc_macro_attribute` Defines an attribute macro.
  - Diagnostics
    - `allow`, `warn`, `deny`, `forbid` Alters the default lint level.
    - `deprecated` Generates deprecation notices.
    - `must_use` Generates a lint for unused values.
    - `diagnostic::on_unimplemented` Hints the compiler to emit a certain
      error message if a trait is not implemented.
  - ABI, linking, symbols, and FFI
    - `link` Specifies a native library to link with an extern block.
    - `link_name` Specifies the name of the symbol for functions or statics in
      an extern block.
    - `link_ordinal` Specifies the ordinal of the symbol for functions or
      statics in an extern block.
    - `no_link` Prevents linking an extern crate.
    - `repr` Controls type layout.
    - `crate_type` Specifies the type of crate (library, executable, etc.).
    - `no_main` Disables emitting the main symbol.
    - `export_name` Specifies the exported symbol name for a function or
      static.
    - `link_section` Specifies the section of an object file to use for a
      function or static.
    - `no_mangle` Disables symbol name encoding.
    - `used` Forces the compiler to keep a static item in the output object
      file.
    - `crate_name` Specifies the crate name.
  - Code generation
    - `inline` Hint to inline code.
    - `cold` Hint that a function is unlikely to be called.
    - `no_builtins` Disables use of certain built-in functions.
    - `target_feature` Configure platform-specific code generation.
    - `track_caller` Pass the parent call location to
      `std::panic::Location::caller()`.
    - `instruction_set` Specify the instruction set used to generate a
      functions code
  - Documentation
    - `doc` Specifies documentation. See The Rustdoc Book for more
      information. Doc comments are transformed into `doc` attributes.
  - Preludes
    - `no_std` Removes `std` from the prelude.
    - `no_implicit_prelude` Disables prelude lookups within a module.
  - Modules
    - `path` Specifies the filename for a module.
  - Limits
    - `recursion_limit` Sets the maximum recursion limit for certain
      compile-time operations.
    - `type_length_limit` Sets the maximum size of a polymorphic type.
  - Runtime
    - `panic_handler` Sets the function to handle panics.
    - `global_allocator` Sets the global memory allocator.
    - `windows_subsystem` Specifies the windows subsystem to link with.
  - Features
    - `feature` Used to enable unstable or experimental compiler features. See
      The Unstable Book for features implemented in rustc.
  - Type System
    - `non_exhaustive` Indicate that a type will have more fields/variants
      added in future.
  - Debugger
    - `debugger_visualizer` Embeds a file that specifies debugger output for a
      type.
    - `collapse_debuginfo` Controls how macro invocations are encoded in
      debuginfo.
- 7.1. Testing
- 7.2. Derive
- 7.3. Diagnostics
- 7.4. Code generation
- 7.5. Limits
- 7.6. Type System
- 7.7. Debugger

## The Reference - Chapter 8. Statements and expressions

- 8.1. Statements
- 8.2. Expressions
  - Type cast expressions
    - `as` supports all allowed coercions, plus more
    - numeric to numeric
      - e.g., `i32` to `f32`, `i64` to `i32`, etc.
    - enum to integer (only when the enum is plain)
    - `bool` or `char` to integer
    - `u8` to `char`
    - `*T` to `*U` (reinterpret cast)
    - `*T` to integer (can truncate)
    - integer to `*T`
    - `&T` to `*T` (but no const cast)
    - `&[T; n]` to `*T` (array to pointer)
    - function item to `*T` (because the type does not matter) or integer
    - function pointer to `*T` (because the type does not matter) or integer
    - non-capturing closures to function pointers

## The Reference - Chapter 9. Patterns

## The Reference - Chapter 10. Type system

- 10.1. Types
  - built-in types
    - primitive types
      - boolean: `bool`
      - numeric: `{u,i}{8,16,32,64,128}` and `f{32,64}`
      - textual: `char` and `str`
      - never: `!`
        - can only be used as the return type (of a noreturn func)
    - sequence types
      - tuple: `(T...)`
        - `()` is allowed and is a zero-sized type (ZST)
          - it is called unit
      - array: `[T; N]`
      - slice: `[T]`
        - dynamically-sized type (DST)
          - not `Sized` and cannot be instantiated
          - this is because the slice len is unknown
        - instantiated as pointer types such as `&[T]`
          - pointer types to DST are `Sized`, but is twice the size of pointer
            types to `Sized`
          - also known as fat pointers
          - the first pointer points to the underlying data
          - the second pointer stores the slice len in-place
        - `str` is `[u8]` whose data are guaranteed to be valid utf8
    - user-defined types
      - struct: `struct Name ...`
        - `struct Name;` or `struct Name{};` are allowed and are ZST
        - `struct Name(T...)` is allowed and is called a tuple struct
      - enum: `enum Name ...`
        - each enum item is like a struct, including empty struct
      - union: `union Name ...`
        - always unsafe to read
    - function types
      - function items (regular functions): `fn name ...`
        - always ZST
        - always implements
          - `Fn` (implies `FnMut` and `FnOnce`)
          - `Copy` (implies `Clone`)
          - `Send` and `Sync`
      - closures: `|...| ...`
        - a closure is approximately a struct
          - the captured variables become struct fields
          - it implements at least `FnOnce`, where the implementation is the
            closure body
          - it may implement `FnMut` or `Fn`, depending on the closure body
    - pointer types
      - references: `&T` and `&mut T`
      - raw pointers: `*const T` and `*mut T`
        - always unsafe to dereference
      - function pointers: `fn ...`
        - say we have two functions, `fn test1(){}` and `fn test2(){}`
        - the two functions are different types
          - `let mut a = test1; a = test2;` fails
          - `let mut a = &test1; a = &test2;` fails
        - the two functions can be coerced to the same function pointer type
          - `let mut a: fn(); a = test1; a = test2; a = ||{}` works
            - because functions and non-capturing closures can be coerced to
              function pointers
          - `let mut a: fn(); a = &test1;` fails
            - because a ref to a function is not the same nor cocerced to a
              function pointer
        - always implements `Fn` (implies `FnMut` and `FnOnce`)
    - trait types
      - trait objects: `dyn Trait...`
        - dynamically-sized type (DST)
          - not `Sized` and cannot be instantiated
          - this is because the concrete type is unknown
        - instantiated as pointer types such as `&dyn Trait`
          - pointer types to DST are `Sized`, but is twice the size of pointer
            types to `Sized`
          - also known as fat pointers
          - the first pointer points to an instance of the concrete type
          - the second pointer points to a vtable
        - note that a trait itself, `trait Trait ...`, is not a type
      - impl trait: `impl Trait`
        - `fn foo(arg: impl Trait)` is approximately
          `fn foo<T: Trait>(arg: T)`, but different
        - can only be used as the param or return type of a func
- 10.2. Dynamically Sized Types
- 10.3. Type layout
- 10.4. Interior mutability
- 10.5. Subtyping and Variance
- 10.6. Trait and lifetime bounds
- 10.7. Type coercions
  - implicit type conversions
  - allowed type coercions
    - `T` to `U`, if `T` is a subtype of `U`
      - this usually means `T` and `U` are the same type with different
        lifetimes
      - e.g., `&'static str` is a subtype of `&str`
    - `T1` to `T3`, if `T1` coerces to `T2` and `T2` coerces to `T3`
    - `&mut T` to `&T`
    - `*mut T` to `*const T`
    - `&T` to `*const T`
    - `&mut T` to `*mut T`
    - (`&T` or `&mut T`) to `&U`, if `impl Deref for T { type Target = U; }`
      - this usually means `T` wraps `U` and can be used as `U`
      - e.g., `Box<T>` derefs to `&T`, `String` derefs to `str`
    - `&mut T` to `&mut U`, if `impl Deref for T { type Target = U; }`
    - `TyCtor(T)` to `TyCtor(U)`, if `T` unsized-coerces to `U`
      - `TyCtor(T)` is one of
        - `&T`, `&mut T`
        - `*const T`, `*mut T`
        - `Box<T>`
      - unsized coercions convert sized types to unsized types, such as
        - `[T; n]` to `[T]`
        - `T` to `dyn U`, if `T` implements `U + Sized` and `U` is
          object safe (i.e., can be represented by a vtable)
    - function item types (regular functions) to `fn` pointers
    - non-capturing closures to `fn` pointers
    - `!` to `T`
  - there is also the type cast operator, `as`
- 10.8. Destructors
- 10.9. Lifetime elision

## The Reference - Chapter 11. Special types and traits

## The Reference - Chapter 12. Names

- 12.1. Namespaces
- 12.2. Scopes
- 12.3. Preludes
- 12.4. Paths
- 12.5. Name resolution
- 12.6. Visibility and privacy

## The Reference - Chapter 13. Memory model

- 13.1. Memory allocation and lifetime
- 13.2. Variables

## The Reference - Chapter 14. Linkage

- each compilation processes a crate in the source form and produces the
  crate in binary form(s)
- `--crate-type=<type>` or `#![crate_type = "<type>"]` specifies the binary
  form(s)
  - `bin` produces an executable
    - there must be a `main` function
  - `lib` produces a library
    - it aliases one of other types below, determined by the compiler
  - `dylib`/`rlib` produces a dynamic/static rust library
    - it contains metadata and can be used as a dep by other crates
  - `cdylib`/`staticlib` produces a dynamic/static system library
    - it is for ffi
  - `proc-macro`
- dependencies
  - for `staticlib`, deps must be `rlib`
  - for `rlib`, deps can be `rlib` or `dylib`
    - `rlib` references its deps and does not own copies of its deps
  - for `bin`, `rlib` is attempted first before `dylib`
  - for `dylib`, deps can be `rlib` or `dylib`
    - `rlib` owns copies of its deps
- statically-linked executables
  - both static linking and dynamic linking of C runtime may be supported
    - it depends on the targets
    - `x86_64-unknown-linux-gnu` defaults to dynamic linking
    - `x86_64-unknown-linux-musl` defaults to static linking
  - `-C target-feature=[+-]crt-static` to enables/disables static linking

## The Reference - Chapter 15. Inline assembly

## The Reference - Chapter 16. Unsafety

- 16.1. The unsafe keyword
- 16.2. Behavior considered undefined
- 16.3. Behavior not considered unsafe

## The Reference - Chapter 17. Constant Evaluation

## The Reference - Chapter 18. Application Binary Interface

## The Reference - Chapter 19. The Rust runtime

## The Reference - Chapter 20. Appendices

- 20.1. Macro Follow-Set Ambiguity Formal Specification
- 20.2. Influences
- 20.3. Glossary

## API Guidelines

- <https://rust-lang.github.io/api-guidelines/>
- Sec 1. Naming
  - when camel casing, `Uuid`, `Usize`, `Stdin`, etc.
  - ad-hoc conversoins
    - `as_*` is free and is between refs
      - it conceptually returns a view of the underlying type
      - all string types have `as_*`
    - `to_*` has a cost and can fail
      - it conceptually converts from one type to another
      - `str`, `OsStr`, and `cstr` have `to_*`
      - `String`, `OsString`, and `CString` do not
    - `into_*` consumes self
      - it conceptually reuses the underlying storage
      - `String`, `OsString`, and `CString` have `into_*`
      - `str`, `OsStr`, and `cstr` do not (unless they are in `Box`)
      - if the type is a wrapper to another, `into_inner` returns the other
        type
  - getters do not have `get_` prefix
    - unless the type has only a single getter, then it is called `get`
  - a collection type should have `iter`, `iter_mut`, and `into_iter`
  - feature names are concise, `foo` instead of `with_foo`
- Sec 2. Interoperability
  - implements all common traits that make sense
    - `Default`
      - if `Default` makes sense, `new` with no arg makes sense too
    - `Copy`, `Clone`
    - `Debug`, `Display`
    - `Eq`, `PartialEq`, `Ord`, `PartialOrd`
    - `Hash`
    - because of the orphan rule, a client cannot implement common traits for
      types defined by a crate
  - implements all conversion traits that make sense
    - `From`, `TryFrom`
      - never implements `Into` or `TryInto`, as they are implemented via
        `From` or `TryFrom`
      - `?` operator inserts `from` so this is especially useful for error
        types
    - `AsRef`, `AsMut`
      - e.g., a function that accepts a path should be
        `fn foo(path: impl AsRef<Path>)` to accept as many types as possible
  - implements `FromIterator` and `Extend` for collections
    - to enable `Iterator::collect`, `Iterator::partition`, etc.
  - implements `Serialize` and `Deserialize` from `serde` for types that hold
    pure data
  - implements `Send` and `Sync` whenever possible
  - implements `Error` for error types
    - also `Send` and `Sync`
    - also `Debug` and `Display`
  - implements `std::fmt::*` such as `std::fmt::LowerHex` for bitflag types
  - takes `T: Read` or `T: Write` params by values
    - it will work with `T` or `&mut T`, as std implements the trait for `&mut
      T`
- Sec 3. Macros
- Sec 4. Documentation
- Sec 5. Predictability
  - no non-static inherent methods on smart pointers
    - `impl Foo { fn bar(p: Self) {} }` instead of
      `impl Foo { fn bar(self) {} }`
    - it enforces the use of `Foo::bar(p)` rather than `p.bar()`
    - it seems
      - `impl Foo { ... }` are inherent methods
        - if a method takes `self` as the first arg, it is non-static
      - `impl Bar for Foo { ... }` are trait methods
  - define conversion methods on the more specific type if there is one
    - e.g., we can convert between `str` and `[u8]` and `str` is more specific
      - conversion methods are defined for `str`, such as `str::as_bytes` or
        `std::str::from_utf8`
    - when defining methods, prefer `to_*`/`as_*`/`into_*` over `from_*`
      because `val.to_foo()` is more ergonomic than `Foo::from(val)`
  - when a function acts on a type, make it a method
  - prefer return vals over out params
    - prefer `fn foo() -> Out` over `fn foo(&mut Out)`
    - this does not apply when `foo` modifies the data than creating the data
  - no surprise with `std::op` traits
    - they are for operator overloading and should not have any surprise
    - implements `std::ops::Deref` only for smart pointers
      - because it participates in implicit type coercion and method
        resolution
      - that's how `&String` is coerced to `&str`, or how `val.foo()` works
        for both `&T` and `Box<T>`, transparently
  - no surprise with `std::op` traits
  - ctors
    - `new` takes no or obvious params
      - it usually does not fail, but it can
      - `String::new` and `OsString::new` take no param
      - `CString::new` and `OsStr::new` take a generic param
        - `CString::new` can fail
      - `str` and `CStr` have no `new`
    - `with_<variant>` (and `new_with_<variant>`) are variants of `new`
      - `String::with_capacity` and `OsString::with_capacity`
      - consider the builder pattern instead
    - `from_<another>` converts from another type
      - it usually takes ownership, but it may not
      - it is more flexible than the `From` trait
        - it can fail and it can take multiple params
      - `String`, `OsString`, `CString`, `OsStr`, and `CStr` have `from_*`
      - `str` does not
    - impl `Default` trait if makes sense
      - all string types implement the `Default` trait
      - it is often expected for both `new` and `Default` to exist and to
        behave the same
    - impl `From` trait if makes sense
      - it usually takes ownership, but it may not (e.g., `T` is a ref)
      - `String`, `OsString`, and `CString` implements `From`
      - `str`, `OsStr`, and `CStr` do not
- Sec 6. Flexibility
  - if intermediate results of a function are interesting, return them as well
    - e.g., `Vec::binary_search` does not return just a bool or the idx
  - function param ownership
    - if a func requires ownership of a param, use `Foo`
      - do not clone in the func but let the caller decides
    - if a func does not require ownerhsip, use `&Foo` or `&mut Foo`
  - use generics to minimize assumptions on func params
    - e.g., `fn foo(path: impl AsRef<Path>)` rather than `fn foo(path: &Path)`
  - decide if a trait is to be used as a trait bound or a trait object or both
    - a trait is used as a trait bound in generics such as `T: Trait`
      - it means `T` is any type that implements `Trait`
    - a trait is used as a trait object types such as `dyn Trait`
      - `dyn Trait` is itself a type
      - it is a DST and cannot be instantiated
      - use `Box<dyn Trait>` instead
    - generics vs trait object
      - static dispatch vs dynamic dispatch
        - generics wins
      - regular pointer vs fat pointer
        - generics wins
      - homogeneous type vs heterogeneous types
        - generics is not an option...
    - trait object limits
      - trait objects are implemented via vtables
      - the compiler needs to generate vtables and all methods in the vtable
        must be fully known at compile-time
      - the traits cannot have generics at all
        - the trait itself can be generic, `trait Foo<T>`
        - but it cannot have generic methods, `trait Foo { fn bar<T>(t: T); }`
        - this is also allowed,
          - `trait Foo { type Bar; }` and `dyn Foo<Bar = Xxx>`
      - the traits also cannot have `Self` other than as the first param
        - because `Self` is like generics and its concrete type is unknown at
          compile-time
        - the first param is special because it is always a pointer to the
          object in the vtable
        - this can be worked around by excluding such methods from the vtable
          with `Self: Sized` bound
    - trait object examples
      - `std::io::Read` and `std::io::Write` are often used as trait objects
      - `std::iter::Iterator` can be used as trait objects
      - note how they all exclude some methods from vtables with `Self: Sized`
- Sec 7. Type safety
  - newtype pattern
    - e.g., `struct Modifier(u64)` and uses `Modifier` for type safetyp
  - use enum or others in place of `bool` in func params
    - e.g., `sync(Start)` is better than `sync(true)`
  - use `bitflags` instead of `enum` for bitflags
  - builder pattern
    - non-consuming builders, such as `std::process::Command`
      - the config methods are `fn foo(&mut self...) -> &mut Self`
      - the terminal methods are `fn build(&self)`
    - consuming builders
      - the config methods are `fn foo(mut self...) -> Self`
      - the terminal methods are `fn build(self)`
- Sec 8. Dependability
  - validate func args
    - static enforcement, such as the newtype pattern
    - dynamic enforcement, with runtime overhead
      - two common ways to avoid the overhead are
        - use `debug_assert!` to enable it for debug builds
        - add `foo_unchecked` variant or `raw` submodule
  - drop never fails
    - if cleanup can fail or can block, define a method for explicit cleanup
    - if the explicit cleanup is not called, drop should eat and log the
      failure
- Sec 9. Debuggability
  - `Debug` should be implemented for all public types
  - `Debug` should output something even the val is empty, such as `""` for
    empty str
- Sec 10. Future proofing
  - sealed traits to prevent clients from implementing the traits
  - keep struct fields private unless must
    - making a field public is a strong commitment
    - structs with private fields cannot be constructed with literals
  - return `impl Trait` if that's sufficient
    - e.g., return `impl Iterator<Item = u32>` allows clients to use the
      return value as an iterator while allowing us to change the concrete
      type at anytime
    - this can also be achieved with newtype pattern
  - do not duplicate derives and trait bounds for plain structs
    - `#[derive(Debug)] struct Foo<T>` is fine
      - adding another derive does not break clients
      - if `T` does not implement `Debug`, neither will Foo
    - `#[derive(Debug)] struct Foo<T: Debug>` is unnecessary and bad
      - adding another derive _and_ bound can break clients, because of the new
        bound requirement
- Sec 11. Necessities
- semver compat
  - <https://doc.rust-lang.org/cargo/reference/semver.html>
- api evolution
  - <https://rust-lang.github.io/rfcs/1105-api-evolution.html>

## Rustonomicon

- <https://doc.rust-lang.org/stable/nomicon/>

## Ownership

- each value in Rust has a variable that is the owner of the value
- there can only be one owner for a value at any time
- when the owner goes out of scope, the value is dropped
- ownership can be transferred from one variable to another
  - `let a = String::from("hello"); let b = a;`
  - b becomes the owner of the String
- when a variable is mutable, it can own different values at different times
  - `let mut a = 3; a = 4`
- simple types, where ownership transfer may be more expensive than copying,
  usually implement Copy trait
  - `let a = 3; let b = a;`
  - both a and b own different values
  - no ownership transfer
- ownerhsip eliminates one major issue: the value of a variable can be
  modified behind the back of the variable
  - no more dangling pointer like in C/C++

## Borrowing

- References for a variable can be created.  They "borrow" the onwership from
  the variable.
  - there are also "unsafe" pointers for a variable
- a variable's ownership of a value can be immutably borrowed
  - `let a = 3; let b = &a; let c = &a;`
  - there can be multiple immutable borrowers
  - until b and c go out of scope, a's ownership is immutably borrowed.  a can
    still be used, but its ownership of the value cannot be transferred
- a variable's ownership of a value can also be mutably borrowed
  - `let mut a = 3; let b = &mut a;`
  - there can only be one mutable borrower
  - until b goes out of scope, a's ownership is mutably borrowed.  a cannot be
    used.
- a variable cannot be borrowed mutably and immutably at the same time
- Internally, a reference is a pointer to the original value.  The borrowing
  rules make sure there is no surprise

## Arrays and Slices

- arrays have type `[T; N]`; different Ns mean different types
- slices have type `[T]`
  - slices have unknown size.  Similar to `str` or `Path`, there can only be
    references to slices
  - a slice borrows from an aray (or some collections)
- vectors implements trait `Deref<Target = [T]>`, meaning they can be treated
  like slices
  - looping vectors in for-loops moves because of `into_iter`
  - looping slices in for-loops copies because references implement Copy trait
- Internally, a slice is two pointers, delimiting a region in the original
  value.  The borrowing rules maek sure there is no surprise.
  - this is why slices are for types that are contiguous in memory

## Pattern Matching

- when a pattern has a match, a variable binding can be optionally created
  - `Some(val)` creates a variable binding which becomes the owner of the
    value (unless the type has Copy trait)
  - `Some(_)` does not
  - `Some(mut val)` creates a mutable variable binding
  - `Some(ref val)` creates a reference
  - `Some(ref mut val)` creates a mutable reference
  - `Some(&val)` can match `Option<&T>` and val becomes the owner of `T`
    - T should be a reference itself because we cannot move a borrowed
      ownership

## Iterators

- a for-loopable variable must implement IntoIter trait
  - the trait allows a variable to be turned into a iterator and transfers the
    ownership to the iterator
- a for-loop is a syntax sugar that turns the looped variable into an iterator
  - `for v in var` becomes roughly
    - `let mut iter = var.into_iter();`
    - `while let v = Some(var.next())`

## Interior Mutability

- interior mutability is similar to a mutable data member in a C++ class
- `UnsafeCell<T>` is T, but provides a get() method to return `*mut T`
  - it is unsafe (because a pointer to T does not borrow the ownership)
  - there is no additional cost
- `Cell<T>` is T with the interior mutability
  - `let a = Cell::new(3); a.set(4); let b = a.get();`
  - it is implemented on top of `UnsafeCell<T>`
  - it is safe because get() returns a copy of T
  - there is no additional cost
- `RefCell<T>` is T with the interior mutability
  - `let a = RefCell::new(3); *a.borrow_mut() = 4; let b = a.borrow();`
  - there is a cost to track the borrow counts dynamically

## Smart Pointers

- `Box<T>` is T put on the heap
  - T's size can be unknown (e.g., recursive types)
    - `struct List<T> { node: T, next: Box<List<T>> }`
  - like `std::unique_ptr<T>`
- `Rc<T>` is T put on the heap together with a refcount
  - like `std::shared_ptr<T>`
- `Arc<T>` is T put on the heap together with an atomic refcount

## Threads

- most types implements Send trait
  - it means the ownership of a value of the type can be transferred to
    another thread
- most types implements Sync trait
  - it means a value of the type can be referenced by multiple threads
  - T is Sync iff &T is Send
- `Rc<T>` implements neither
  - when a `Rc<T>` is shared by two threads, the refcount can be messed up
    refcount can be messed up
- `Arc<T>` implements both

## The Rust Standard Library

- <https://doc.rust-lang.org/std/>
  - there is also <https://doc.rust-lang.org/core/>
    - core provides stuff that has no external dependency (such as libc,
      system libs, etc)
    - it is usable in `no_std` cfg
    - I guess `std` re-exports `core`
- macros
  - `assert`, `assert_eq`, and `assert_ne` assert
  - `cfg` evaluates cfg flags at compile time
  - `file`, `line`, and `column` expand to loc in the source file
  - `compile_error` generates a compile error
  - `concat` concatenates literals into `&'static str`
  - `dbg` prints and returns the value of an expression for quick debug
  - `debug_assert`, `debug_assert_eq`, and `debug_assert_ne` are asserts that
    are disabled in optimized builds by default
  - `env` and `option_env` return an envvar at compile time
  - `format` returns a `String`
  - `panic`, `todo`, `unimplemented`, `unreachable` all panic
  - `print` and `println` print to stdout
  - `eprint` and `eprintln` print to stderr
  - `write` and `writeln` print to a buffer
  - `stringify` returns `&'static str`
  - `thread_local` makes static decls thread-local
  - `vec` returns a `Vec`
- modules
  - `alloc` provides unsafe memory allocation
  - `any` is akin to C++ RTTI
  - `arch` provides arch-dependent cpu intrinsics
  - `array` provides utils to work with arrays
  - `ascii` provides utils for ascii (rather than utf8) strings
  - `backtrace` captures a stack backstrace
  - `borrow` provides traits such as `Borrow` or `ToOwned`
  - `boxed` provides `Box<T>`
  - `char`, `f32`, and `f64` should be considered internal
  - `cell` provides `Cell<T>` and variants
  - `clone` provides the `Clone` trait
  - `cmp` provides traits such as `Eq` or `Ord`, as well as utils
  - `collections` provides collection types
    - sequences: `Vec`, `VecDeque`, `LinkedList`
    - maps: `HashMap`, `BTreeMap`
    - sets: `HashSet`, `BTreeSet`
    - misc: `BinaryHeap`
  - `convert` provides traits such as `AsMut`, `AsRef`, `From`, `Into`, etc.
    - between `AsRef` and `From`
      - `AsRef` is essentially nop, e.g., `str` implements `AsRef<OsStr>`
      - `From` can have a cost, e.g., `String` implements `From<&str>`
  - `default` provides the `Default` trait
  - `env` works with process env, such as envvars, args, etc.
  - `error` provides the `Error` trait
  - `ffi` provides ffi bindings
    - c types: `c_void`, `c_char`, `c_int`, etc.
    - os strings: `OsStr`, `OsString`
      - rust strings, `str` and `String`, always store valid utf8 with
        explicit size
        - it does not rely on `\0` to terminate
        - `\0` is a valid utf8 char and can appear anywhere
        - there is no `\0` in the end
      - os strings, `OsStr` and `OsString`, always store valid wtf8 with
        explicit size
        - wtf8 is a superset of utf8, and as a result,
          - rust strings are os strings, and casting has no cost
          - os strings are not necessarily rust strings
            - casting requires no allocation but requires validation
        - this is done because the os can accept strings that are not valid utf8
        - on win, the os expects utf16 so it takes another conversion from os
          strings to to what win expects
    - c strings: `CStr`, `CString`
      - `CStr` and `CString` store any non-`\0` value and are `\0`-terminated
      - conversions between rust strings can fail in both directions
  - `fmt` provides string formatting traits and utils
    - `{}` formats with `Display` trait
    - `{:?}` formats with `Debug` trait
    - `{:x?}` formats with `Debug` trait, lower-case hex
    - `{:X?}` formats with `Debug` trait, upper-case hex
    - `{:o}` formats with `Octal` trait
    - `{:x}` formats with `LowerHex` trait
    - `{:X}` formats with `UpperHex` trait
    - `{:p}` formats with `Pointer` trait
    - `{:b}` formats with `Binary` trait
    - `{:e}` formats with `LowerExp` trait
    - `{:E}` formats with `UpperExp` trait
  - `fs` provides cross-os filesystem support
  - `future` is for `async` and `await`
  - `hash` provides traits such as `Hash`, `Hasher`, etc.
  - `hint` provides compiler hints
  - `io` provides cross-os io support
    - traits such as `Read`, `Write`, and `Seek`
      - also `BufRead` (read to an internal buffer first) and `IsTerminal`
    - funcs such as `stdin`, `stdout`, `stderr`
    - `Error` enum
  - `iter` provides traits and utils for iterations
    - traits such as `Iterator`, `DoubleEndedIterator`, and `ExactSizeIterator`
      - also `IntoIterator`, `FromIterator`
  - `marker` provides traits and types for basic props of types
    - traits such as `Copy`, `Send`, `Sized`, `Sync`,
    - `PhantomData`
  - `mem` provides utils about memory
    - `offset_of` macro
    - funcs such as `align_of`, `size_of`, `drop`, `swap`, `take`, `replace`
  - `net` provides cross-os tcp/udp support
  - `num` provides utils for numerical computation
  - `ops` provides operator overloading
    - traits such as `Add`, `Mul`, `Drop`, `Deref`, etc.
      - also `Fn`, `FnMut`, and `FnOnce` for callbacks
        - there are 3 kinds of func-like types
          - func items (that is, regular funcs)
          - closures
          - func pointers
        - they implement one or more of the traits
  - `option` provides `Option<T>`
  - `os` provides OS-specific utils
    - `linux` submodule, which is kinda empty
    - `unix` submodule, which extends `ffi`, `fs`, `io`, `net`, `process`,
      `thread`, etc.
    - `fd` submodule
      - types such as `RawFd`, `BorrowedFd`, `OwnedFd`
      - traits such as `AsFd`, `AsRawFd`, `FromRawFd`, `IntoRawFd`
    - `raw` submodule, which has been replaced by `core::ffi`
  - `panic` provides panic support
  - `path` provides cross-os path support
    - types such as `PathBuf` and `Path`
  - `pin` pins data to fixed addr
  - `prelude` uses a subset of std automatically
    - rust 2018
      - `std::borrow::ToOwned`
      - `std::boxed::Box`
      - `std::clone::Clone`
      - `std::cmp::{PartialEq, PartialOrd, Eq, Ord}`
      - `std::convert::{AsRef, AsMut, Into, From}`
      - `std::default::Default`
      - `std::iter::{Iterator, Extend, IntoIterator, DoubleEndedIterator, ExactSizeIterator}`
      - `std::marker::{Copy, Send, Sized, Sync, Unpin}`
      - `std::mem::drop`
      - `std::ops::{Drop, Fn, FnMut, FnOnce}`
      - `std::option::Option::{self, Some, None}`
      - `std::result::Result::{self, Ok, Err}`
      - `std::string::{String, ToString}`
      - `std::vec::Vec`
    - rust 2021
      - `std::convert::{TryFrom, TryInto}`
      - `std::iter::FromIterator`
    - other modules might have `prelude` submodules that can be used manually
      - `std::io::prelude`
      - `std::os::unix::prelude`
  - `primitive` re-exports primitive types which can be useful for
    macro-generated code
  - `process` provides cross-os process support
    - structs such as `Command` and `Child`
    - funcs such as `abort` and `exit`
  - `ptr` provides mostly unsafe utils for raw pointers
  - `rc` provides `Rc<T>` and `Weak<T>`
  - `result` provides `Result<T>`
  - `slice` provides utils to slice
  - `str` provides utils to `str`
  - `string` provides `String`
  - `sync` provides structs such as `Arc`, `Mutex`, etc.
  - `task` provides structs and traits for async tasks
  - `thread` provides cross-os native thread support
    - types such as `Builder`
    - funcs such as `current`, `park`, `sleep`, `spawn`, `yield_now`
  - `time` provides structs such as `Duration`, `Instant` (`CLOCK_MONOTONIC`),
    `SystemTime` (`CLOCK_REALTIME`)
  - `vec` provides `Vec<T>`
