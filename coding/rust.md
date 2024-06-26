Rust
====

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

- `rustup target add aarch64-unknown-linux-gnu`
- `cargo build --target aarch64-unknown-linux-gnu`
- edit `~/.cargo/config` to add
    [target.aarch64-unknown-linux-gnu]
    linker = "aarch64-linux-gnu-gcc"

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
  - `impl Foo {...}` defines methods for `Foo`

## The Book - Chapter 6. Enums and Pattern Matching

- 6.1. Defining an Enum
- 6.2. The match Control Flow Construct
- 6.3. Concise Control Flow with if let

## The Book - Chapter 7. Managing Growing Projects with Packages, Crates, and Modules

- 7.1. Packages and Crates
- 7.2. Defining Modules to Control Scope and Privacy
- 7.3. Paths for Referring to an Item in the Module Tree
- 7.4. Bringing Paths Into Scope with the use Keyword
- 7.5. Separating Modules into Different Files
  - we can build a module tree in a single file
  - we can also build a module tree using multiple files and directories
  - cargo knows `src/main.rs` or `src/lib.rs` is the crate root
    - `mod foo` in the crate root tells cargo to build `src/foo.rs`
    - `mod bar` in `src/foo.rs` tells cargo to build `src/foo/bar.rs`

## The Book - Chapter 8. Common Collections

- 8.1. Storing Lists of Values with Vectors
- 8.2. Storing UTF-8 Encoded Text with Strings
- 8.3. Storing Keys with Associated Values in Hash Maps

## The Book - Chapter 9. Error Handling

- 9.1. Unrecoverable Errors with panic!
- 9.2. Recoverable Errors with Result
- 9.3. To panic! or Not to panic!

## The Book - Chapter 10. Generic Types, Traits, and Lifetimes

- 10.1. Generic Data Types
- 10.2. Traits: Defining Shared Behavior
  - `trait Foo { fn bar(&self)... }` defines trait `Foo`
  - `impl Foo for Baz` implements trait `Foo` for type `Baz`
- 10.3. Validating References with Lifetimes

## The Book - Chapter 11. Writing Automated Tests

- 11.1. How to Write Tests
- 11.2. Controlling How Tests Are Run
- 11.3. Test Organization

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

- 15.1. Using Box<T> to Point to Data on the Heap
- 15.2. Treating Smart Pointers Like Regular References with the Deref Trait
- 15.3. Running Code on Cleanup with the Drop Trait
- 15.4. Rc<T>, the Reference Counted Smart Pointer
- 15.5. RefCell<T> and the Interior Mutability Pattern
- 15.6. Reference Cycles Can Leak Memory

## The Book - Chapter 16. Fearless Concurrency

- 16.1. Using Threads to Run Code Simultaneously
- 16.2. Using Message Passing to Transfer Data Between Threads
- 16.3. Shared-State Concurrency
- 16.4. Extensible Concurrency with the Sync and Send Traits

## The Book - Chapter 17. Object Oriented Programming Features of Rust

- 17.1. Characteristics of Object-Oriented Languages
- 17.2. Using Trait Objects That Allow for Values of Different Types
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
- 21.4. D - Useful Development Tools
- 21.5. E - Editions
- 21.6. F - Translations of the Book
- 21.7. G - How Rust is Made and “Nightly Rust”

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
